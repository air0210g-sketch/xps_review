# Redis 写入过程流程图

> 从**网络 IO**（接收写请求）→ **数据写入**（命令执行、写内存）→ **数据持久化**（AOF / RDB）的完整流程。与 [[20250303_02_Redis单线程模型_3W1H解剖]]、[[20250303_Atomic_持久化RDB与AOF]] 配合阅读。

---

## 一、写入过程总览（活动图）

采用**活动图**表达阶段划分与数据流（符合 [plantuml_use](https://plantuml.com/zh/activity-diagram-beta) 中「阶段划分、数据流、分支流程 → 活动图」）。

```plantuml
@startuml
!theme mars
title Redis 写入过程（按处理线程着色）

legend right
  |= 颜色 |= 线程/进程 |
  |<#E3F2FD>   | 主线程 |
  |<#FFF3E0>   | IO 线程（6.0+） |
  |<#E8F5E9>   | 后台/bio 线程 |
  |<#F3E5F5>   | 子进程（RDB / AOF 重写） |
  |= 线型 |= 含义 |
  | 虚线 | 异步流程（持久化与主请求异步） |
endlegend

start

partition "主线程" #E3F2FD {
  :客户端发送写命令;
  :IO 多路复用检测到 socket 可读;
}

if (Redis 6.0+ 多线程 IO?) then (是)
  partition "IO 线程" #FFF3E0 {
    :从 socket 读取到读缓冲区;
  }
  partition "主线程" #E3F2FD {
    :从读缓冲区取完整命令;
  }
else (否)
  partition "主线程" #E3F2FD {
    :从 socket 读取请求;
  }
endif

partition "主线程" #E3F2FD {
  :解析命令（协议、参数）;
  :查找并执行命令（如 setCommand）;
  :写内存数据结构（dict / 跳表 / SDS 等）;
  :将响应写入写缓冲区;
  :写命令追加到 AOF 缓冲区;
}

if (AOF 开启且 fsync=always?) then (是)
  partition "主线程" #E3F2FD {
    :AOF 缓冲区写入 AOF 文件;
    :fsync 落盘（主线程阻塞直至完成）;
  }
  note right: always 时先落盘再响应
endif

if (6.0+ 多线程 IO?) then (是)
  partition "IO 线程" #FFF3E0 {
    :将写缓冲区内容写出到 socket;
  }
else (否)
  partition "主线程" #E3F2FD {
    :将响应写出到 socket;
  }
endif

partition "主线程" #E3F2FD {
  :客户端收到响应;
}

if (AOF 开启且 fsync=always?) then (否)
  -[#gray,dashed]->
  :异步持久化（everysec/no 或 RDB）;
  fork
    if (AOF 重写触发?) then (否，常规追加)
      partition "后台/bio 线程" #E8F5E9 {
        :AOF 缓冲区写入 AOF 文件;
        if (fsync 策略?) then (everysec)
          :每秒或由 bio 线程 fsync;
        else (no)
          :由 OS 决定何时落盘;
        endif
      }
    else (是，BGREWRITEAOF / 阈值)
    partition "主线程" #E3F2FD {
      :fork 子进程;
    }
    partition "子进程" #F3E5F5 {
      :遍历内存生成新 AOF（最小命令集）;
      :写临时 AOF 文件;
    }
    partition "主线程" #E3F2FD {
      :主进程继续处理请求;
      :新写入同时进 AOF 缓冲区与 rewrite 缓冲区;
    }
    partition "子进程" #F3E5F5 {
      :子进程完成并退出;
    }
    partition "主线程" #E3F2FD {
      :将 rewrite 缓冲区追加到新 AOF;
      :重命名替换旧 AOF 文件;
    }
  endif
fork again
  if (RDB 触发方式?) then (BGSAVE / 定时)
    partition "主线程" #E3F2FD {
      :fork 子进程;
    }
    partition "子进程" #F3E5F5 {
      :遍历内存生成 RDB 快照（COW）;
      :写 dump.rdb 并退出;
    }
    partition "主线程" #E3F2FD {
      :主进程继续处理请求（不阻塞）;
    }
  else (SAVE)
    partition "主线程" #E3F2FD {
      :主进程同步遍历内存生成 RDB;
      :主进程写 dump.rdb;
    }
    note right: SAVE 阻塞主线程\n此期间不处理新请求
  endif
  end fork
endif

stop
@enduml
```

**图中颜色与线程对应**：主线程（浅蓝）、IO 线程 6.0+（浅橙）、后台/bio 线程（浅绿）、子进程 RDB（浅紫）。同一颜色块内的活动由该线程/进程执行。

**AOF fsync=always 时的流程**：配置为 **always** 时，主线程在**返回响应之前**会先把 AOF 缓冲区写入文件并 **fsync 落盘**，阻塞直至完成后再写响应到 socket，因此图中在「写响应」前增加了「AOF 开启且 fsync=always → 主线程写文件 + fsync」分支；**everysec / no** 时仍为先响应、再异步由后台/bio 写文件与 fsync。

AOF 写入：刷盘配置，sync + per second + no

bgrewrite 配置：开启后，触发阈值会fork 子进程：遍历内存生成最小的AOF2 

主进程：rewrite 缓冲区写入AOF2， 用AOF2 替换 AOF文件。

RDB 写入：取决于redis.conf中的 save xx xx配置

1. 如果存在该配置，则默认启用bgsave
2. 不存在：则不会自动生成rdb文件，除非手动触发save命令，或者redis退出前+主从初始化同步+清空redis

---

## 二、持久化（按线程/进程划分）

### 2.1 持久化三场景时序图（alt）

用时序图的 **alt / else** 表示三种持久化场景的互斥分支（符合 [plantuml_use](https://plantuml.com/zh/sequence-diagram) 中「正常/异常调用顺序、交互 → 时序图」）。

```plantuml
@startuml
!theme mars
title 持久化三场景（alt 互斥分支）

participant "主线程" as Main
participant "后台/bio 线程" as Bio
participant "子进程" as Child

Main -> Main: 写命令已追加到 AOF 缓冲区\n（主路径完成后）

alt 【AOF 常规】缓冲区写文件
  alt fsync=always（先落盘再响应）
    Main -> Main: AOF 缓冲区写入 AOF 文件
    Main -> Main: fsync 落盘（主线程阻塞直至完成）
    note right of Main: 完成后才向客户端返回响应
  else fsync=everysec / no（先响应再异步刷盘）
    Main -> Bio: 触发写盘（定时或缓冲区满）
    Bio -> Bio: AOF 缓冲区写入 AOF 文件
    Bio -> Bio: fsync（按策略）
    Bio --> Main: （异步，主线程不等待）
  end

else 【AOF 重写】BGREWRITEAOF 或超阈值
  Main -> Main: fork 子进程
  Main -> Child: 子进程启动
  Child -> Child: 遍历内存生成新 AOF（最小命令集）
  Child -> Child: 写临时 AOF 文件
  Main -> Main: 新写入进 AOF 缓冲区 + rewrite 缓冲区
  Child -> Child: 完成并退出
  Child --> Main: 子进程退出
  Main -> Main: 将 rewrite 缓冲区追加到新 AOF
  Main -> Main: 重命名替换旧 AOF 文件

else 【RDB】BGSAVE / 定时 或 SAVE
  Main -> Main: fork 子进程（BGSAVE）或同步写（SAVE）
  Main -> Child: 子进程启动（BGSAVE 时）
  Child -> Child: 遍历内存生成 RDB 快照（COW）
  Child -> Child: 写 dump.rdb 并退出
  Main -> Main: 主进程继续处理请求（BGSAVE 时不阻塞）
  Child --> Main: 子进程退出（BGSAVE 时）
end

@enduml
```

**说明**：最外层 alt 表示三种持久化场景（可组合）；**【AOF 常规】** 内再用 alt 区分 **always**（主线程先写文件+fsync 再响应）与 **everysec/no**（后台/bio 异步写与 fsync），与主活动图中 always 的流程一致。

---

## 三、阶段说明

| 阶段                      | 说明                                                                                                                                                                                                                                                                                                                       |
| ------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **网络 IO（请求）** | 客户端发写命令；epoll/kqueue 等检测到可读；6.0 起可由 IO 线程读入读缓冲区，主线程再取命令；否则主线程直接读 socket。                                                                                                                                                                                                       |
| **数据写入**        | 主线程**单线程**解析命令、查表执行、更新内存（dict、跳表等）、写响应到写缓冲区。写内存是纯内存操作，无磁盘 IO。                                                                                                                                                                                                      |
| **网络 IO（响应）** | 将写缓冲区内容发回客户端；6.0 可由 IO 线程写出，主线程继续处理下一条。                                                                                                                                                                                                                                                     |
| **数据持久化**      | **AOF**：主线程追加到 AOF 缓冲区；**fsync=always** 时主线程在**响应前**写文件并 fsync（阻塞），everysec/no 时由后台/bio 异步写文件与 fsync。**AOF 重写**（BGREWRITEAOF 或超阈值）fork 子进程生成新 AOF，主进程追加 rewrite 并替换。**RDB**：BGSAVE/定时 fork 子进程不阻塞，SAVE 同步写阻塞。 |

---

## 四、要点小结

- **单线程执行**：命令解析与执行、内存写入、AOF 缓冲区追加均在**主线程**顺序完成；网络 IO 在 6.0 可多线程，但命令执行仍单线程。
- **持久化不阻塞主路径**：everysec/no 时 AOF 写文件与 fsync 由 bio 或定时做，RDB 由子进程做；**fsync=always 时例外**，主线程在响应前同步写 AOF 并 fsync，会阻塞、拉高延迟。
- **顺序**：**everysec/no** 时为先写内存再响应、再异步持久化；**always** 时为先写内存、再主线程写 AOF+fsync（阻塞）、再响应，图中已用「AOF 开启且 fsync=always」分支体现。
- **SAVE vs BGSAVE**：图中 RDB 分支已区分——**BGSAVE**（或定时触发）fork 子进程做快照，主进程不阻塞；**SAVE** 由主线程同步写 RDB，会阻塞直至写完，生产环境通常用 BGSAVE 或定时规则。
- **AOF 重写**：图中 AOF 分支已区分——**常规追加**：缓冲区写文件 + fsync；**重写触发**（BGREWRITEAOF 或体积超阈值）时 fork 子进程遍历内存生成新 AOF，主进程新写入同时写 rewrite 缓冲区，子进程完成后主进程将 rewrite 缓冲区追加到新文件并替换旧 AOF，主进程不阻塞。
