# NIO 与零拷贝：通用技术说明

> 多路复用 IO（NIO/select-poll-epoll）与零拷贝（Zero-Copy）是操作系统与网络栈的通用能力，在 **Kafka、Nginx、Redis、Netty** 等中间件中广泛使用。本文档从原理与时序图角度说明，不绑定单一组件。

---

## 一、零拷贝（Zero-Copy）

**场景**：将磁盘/内核缓冲区的数据发往网络（例如：服务端读文件并返回给客户端）。传统方式需多次拷贝与上下文切换，零拷贝通过 `sendfile` 等系统调用让数据在内核内完成传递，不经用户态。

### 1.1 传统方式：多次拷贝 + 多次上下文切换

数据需经过**用户态**，共 **4 次数据拷贝**、**4 次上下文切换**。CPU 参与两次「内核↔用户」的数据搬运，占用 CPU 且延迟高。

```plantuml
@startuml
!theme mars
title 传统方式：Disk → Network（4 次拷贝）

participant "磁盘\n(Disk)" as Disk
participant "内核缓冲区\n(Kernel Buffer\n/ PageCache)" as Kernel
participant "应用\n(用户态)" as User
participant "Socket\n缓冲区" as Socket
participant "网卡\n(NIC)" as NIC

Disk -> Kernel : ① DMA 拷贝\n(read 系统调用)
Kernel -> User : ② CPU 拷贝\n(数据到用户空间)
note right of User : 上下文切换\n用户态处理
User -> Socket : ③ CPU 拷贝\n(数据到 Socket 缓冲区)
Socket -> NIC : ④ DMA 拷贝\n(发往网卡)

note over Disk, NIC
 **代价**：4 次拷贝，4 次上下文切换
 用户态经手完整数据 → CPU 与内存带宽占用大
end note
@enduml
```

### 1.2 sendfile 零拷贝

**sendfile(out_fd, in_fd, offset, count)**：数据在**内核内**从文件描述符拷贝到 Socket，再 DMA 到网卡，**不经过用户态**。仅 **2 次 DMA + 少量描述符拷贝**，上下文切换与 CPU 参与大幅减少。

```plantuml
@startuml
!theme mars
title sendfile 零拷贝（读路径）

participant "磁盘 / PageCache" as Disk
participant "内核\n(文件 → Socket)" as Kernel
participant "应用\n(用户态)" as User
participant "网卡\n(NIC)" as NIC

User -> Kernel : sendfile(out, in, offset, count)\n(仅发起调用，不传数据)
Kernel -> Kernel : 内核内：PageCache → Socket Buffer\n(描述符/元数据级别，或 DMA 聚合)
Kernel -> NIC : DMA 拷贝\n(直接发往网卡)

note right of User : 应用不接触数据体\n只做「发起到哪里、发多少」
note over Disk, NIC
 **收益**：数据不经用户态，2 次 DMA
 少拷贝、少上下文切换、CPU 占用低
end note
@enduml
```

### 1.3 对比小结（零拷贝）

| 项 | 传统 read + write | sendfile 零拷贝 |
|----|-------------------|-----------------|
| 数据拷贝次数 | 4 次 | 2 次（DMA） |
| 上下文切换 | 4 次 | 2 次 |
| 用户态是否经手数据 | 是，完整数据 | 否，仅系统调用参数 |
| CPU 占用 | 高（搬运数据） | 低（仅控制逻辑） |

**典型应用**：Kafka Broker 读 .log 发往 Consumer、Nginx 静态文件下发、各类文件下载/流式传输服务。

---

## 二、多路复用 IO（Multiplexed I/O / NIO）

**场景**：服务端同时处理大量连接（多个 socket），若「一连接一线程」在 read() 上阻塞，线程与内存开销大。多路复用通过 **select / poll / epoll**（Linux）或 **kqueue**（BSD/macOS）在一线程上等待「多个 fd 中至少一个就绪」，再只处理就绪的 fd。

### 2.1 传统方式：多线程 + 阻塞 IO（一连接一线程）

每来一个连接就分配一个线程，该线程在 **read()** 上**阻塞**直到有数据。连接多时线程多，上下文切换与栈内存开销大。

```plantuml
@startuml
!theme mars
title 传统方式：一连接一线程、阻塞 read

participant "Client A" as CA
participant "Client B" as CB
participant "Client C" as CC
participant "服务端\n线程A" as TA
participant "服务端\n线程B" as TB
participant "服务端\n线程C" as TC

CA -> TA : 连接
CB -> TB : 连接
CC -> TC : 连接

TA -> TA : read() **阻塞** 等待 A 数据
TB -> TB : read() **阻塞** 等待 B 数据
TC -> TC : read() **阻塞** 等待 C 数据

note over TA, TC
 **代价**：N 连接 = N 线程
 阻塞时线程不释放，上下文切换多、内存占用大
end note

CA -> TA : 有数据
TA -> TA : 被唤醒，处理
TA -> CA : 响应

@enduml
```

### 2.2 多路复用方式：单线程 + Selector（select/poll/epoll）

**一个线程**持有一个 **Selector**，把多个连接的 fd **注册**到内核，通过 **select() / poll() / epoll_wait()** 等待「至少有一个 fd 就绪」。就绪后只处理有事件的连接。**1 个线程可管理大量连接**。

```plantuml
@startuml
!theme mars
title 多路复用方式：单线程 + Selector（poll/epoll）

participant "Client A" as CA
participant "Client B" as CB
participant "Client C" as CC
participant "Selector\n(内核 poll/epoll)" as Sel
participant "服务端\n(单线程)" as Server

CA -> Sel : 注册 Channel A
CB -> Sel : 注册 Channel B
CC -> Sel : 注册 Channel C

Server -> Sel : select() / poll() / epoll_wait()\n**阻塞**直到有就绪事件
note right of Sel : 内核通知：A、C 可读\n（B 暂无数据不通知）

Sel -> Server : 返回就绪集合 {A, C}
Server -> Server : 遍历就绪集：处理 A、C
Server -> CA : 响应 A
Server -> CC : 响应 C

Server -> Sel : 再次等待 … 循环

note over CA, Server
 **收益**：1 线程处理 N 连接
 仅就绪连接占用 CPU，少线程、少切换
end note
@enduml
```

### 2.3 对比小结（多路复用 IO）

| 项 | 传统多线程阻塞 IO | NIO 多路复用（Selector） |
|----|-------------------|--------------------------|
| 线程与连接关系 | 1 连接 1 线程 | 1 线程 N 连接 |
| 阻塞发生位置 | 每线程在 read() 阻塞 | 单线程在 select/poll/epoll_wait 阻塞，内核通知就绪 |
| 上下文切换 | 连接多时切换多 | 与连接数弱相关，主要与就绪事件数相关 |
| 内存占用 | 每线程栈（如 1MB×万连接） | 单线程 + 内核就绪队列，占用小 |

**典型应用**：Kafka Broker、Redis、Nginx、Netty、各类高并发网关与 RPC 框架。

---

## 三、select、poll、epoll 的区别与原理（Linux）

Java NIO 的 **Selector** 在 Linux 上底层会优先用 **epoll**，未满足时退化为 **poll**；在 BSD/macOS 上多为 **kqueue**。下面以 Linux 为例，说明 select / poll / epoll 的机制与差异。

**共同目标**：在一组 fd（如多个 socket）上等待「至少有一个可读/可写」，避免为每个 fd 开一个阻塞线程。

### 3.1 以服务端接收数据为例：三种方式时序图

场景：服务端同时监听多个客户端（A、B、C）的连接，等待其中任意一个有数据到达并读取处理。

---

**（1）select 方式**

每次调用都把完整 fd_set 传入内核，内核轮询所有 fd，返回时改写同一 fd_set；服务端再遍历整个集合找出就绪的 fd。

```plantuml
@startuml
!theme mars
title 服务端接收数据：select 方式

participant "Client A" as CA
participant "Client B" as CB
participant "Client C" as CC
participant "服务端\n(用户态)" as Server
participant "内核\n(select)" as Kernel

Server -> Server : 初始化 read_fds = {fd_A, fd_B, fd_C}
Server -> Kernel : select(nfds, read_fds, ...)\n**拷贝整个 fd_set 入内核**
activate Kernel
Kernel -> Kernel : **轮询** fd_A, fd_B, fd_C\n检查是否可读 (O(n))

CA -> Kernel : 数据到达 (fd_A 可读)
Kernel -> Kernel : 将 fd_A 在 read_fds 中置位\n**改写** read_fds 后返回
Kernel -> Server : 返回，**同一 fd_set 被改写**\n(就绪信息在 read_fds 里)
deactivate Kernel

Server -> Server : **遍历整个 fd_set**\n找出 fd_A 就绪 (O(n))
Server -> Kernel : read(fd_A) 读取数据
Kernel -> Server : 返回 Client A 的数据
Server -> Server : 处理请求并响应

note over Server, Kernel
 **代价**：每次调用拷贝全量 fd_set
 内核轮询所有 fd，用户态遍历整个集合
end note
@enduml
```

---

**（2）poll 方式**

每次调用传入完整 pollfd 数组，内核轮询所有 fd，结果写回各元素的 revents；服务端遍历整个数组找 revents≠0 的 fd。

```plantuml
@startuml
!theme mars
title 服务端接收数据：poll 方式

participant "Client A" as CA
participant "Client B" as CB
participant "Client C" as CC
participant "服务端\n(用户态)" as Server
participant "内核\n(poll)" as Kernel

Server -> Server : 准备 pollfd[] = [{A,events},{B,events},{C,events}]
Server -> Kernel : poll(pollfd[], 3, timeout)\n**拷贝整个 pollfd[] 入内核**
activate Kernel
Kernel -> Kernel : **轮询** fd_A, fd_B, fd_C\n检查是否可读 (O(n))

CB -> Kernel : 数据到达 (fd_B 可读)
Kernel -> Kernel : 将 fd_B 的 revents 置位\n(不破坏 events)
Kernel -> Server : 返回，**写回 pollfd[]**\n(revents 标记就绪)
deactivate Kernel

Server -> Server : **遍历整个 pollfd[]**\n找 revents≠0，得 fd_B (O(n))
Server -> Kernel : read(fd_B) 读取数据
Kernel -> Server : 返回 Client B 的数据
Server -> Server : 处理请求并响应

note over Server, Kernel
 **代价**：每次调用拷贝全量 pollfd[]
 无 1024 限制，但仍是全量轮询 + 全量遍历
end note
@enduml
```

---

**（3）epoll 方式**

先通过 epoll_ctl 把 fd 注册到内核（一次），之后只调用 epoll_wait；内核用事件驱动把就绪 fd 放入链表，返回时只带出就绪的 fd，服务端只遍历就绪集合。

```plantuml
@startuml
!theme mars
title 服务端接收数据：epoll 方式

participant "Client A" as CA
participant "Client B" as CB
participant "Client C" as CC
participant "服务端\n(用户态)" as Server
participant "内核\n(epoll)" as Kernel

Server -> Kernel : epoll_create()
Kernel -> Server : 返回 epfd
Server -> Kernel : epoll_ctl(ADD, fd_A)\nepoll_ctl(ADD, fd_B)\nepoll_ctl(ADD, fd_C)\n**注册一次，内核维护红黑树**
note right of Kernel : 不在此拷贝全量 fd\n仅增删改单条

Server -> Kernel : epoll_wait(epfd, events[], max)\n**阻塞，不传 fd 列表**
activate Kernel
Kernel -> Kernel : 等待就绪事件\n(不轮询，由网卡/协议栈回调)

CC -> Kernel : 数据到达 → fd_C 就绪\n**回调**：fd_C 加入就绪链表
Kernel -> Server : 返回，**仅带出就绪的 fd**\nevents[] = {fd_C}（长度 1）
deactivate Kernel

Server -> Server : **只遍历 events[]**（就绪集合）\n得到 fd_C (O(就绪数))
Server -> Kernel : read(fd_C) 读取数据
Kernel -> Server : 返回 Client C 的数据
Server -> Server : 处理请求并响应

Server -> Kernel : epoll_wait(...) 再次等待
note over Server, Kernel
 **收益**：无每次全量拷贝，无全量轮询\n仅返回就绪 fd，遍历成本与就绪数相关
end note
@enduml
```


### 3.2 select / poll / epoll 文字说明与对比表

**select**

- **接口**：`select(nfds, read_fds, write_fds, except_fds, timeout)`，传入三个 fd_set 位图（可读/可写/异常）和最大 fd 值。
- **原理**：每次调用把当前关心的 fd 集合从用户态拷贝到内核；内核**轮询**所有传入的 fd（O(n)）；返回时内核**改写**传入的 fd_set，用户态再**遍历整个 fd_set** 找出就绪的 fd。
- **特点**：有最大 fd 数量限制（如 `FD_SETSIZE=1024`）；入参出参复用同一块内存，每次调用前需重新初始化；每次调用都拷贝整个 fd 集合。

**poll**

- **接口**：`poll(pollfd[], nfds, timeout)`，传入 pollfd 数组（fd + events + revents）。
- **原理**：每次调用把 pollfd 数组拷贝到内核；内核轮询所有 fd（O(n)）；就绪结果写回 revents，用户态遍历数组找 revents≠0。
- **特点**：无 1024 限制；输入输出分离（events/revents）；仍为每次全量拷贝与全量轮询。

**epoll**

- **接口**：`epoll_create()`、`epoll_ctl(epfd, op, fd, event)`、`epoll_wait(epfd, events[], maxevents, timeout)`。
- **原理**：fd 通过 epoll_ctl **注册一次**，内核维护红黑树；就绪时通过**回调**将 fd 挂入就绪链表，epoll_wait 只返回就绪的 fd，用户只遍历就绪集合；内核不轮询所有 fd，事件驱动。
- **特点**：无每次调用的 fd 集合拷贝；只返回就绪 fd；支持 LT/ET；连接数大时性能优于 select/poll。

**对比小结**

| 项 | select | poll | epoll |
|----|--------|------|-------|
| fd 集合传递 | 每次调用传入/传出 fd_set | 每次调用传入 pollfd[] | 注册用 epoll_ctl，等待用 epoll_wait，不重复传全量 fd |
| 内核实现 | 轮询所有 fd，O(n) | 轮询所有 fd，O(n) | 事件驱动，就绪 fd 入链表，epoll_wait 只取就绪，O(就绪数) |
| 最大 fd 数 | 受 FD_SETSIZE 限制（如 1024） | 理论受资源限制 | 同左，单机可支撑大量连接 |
| 用户态获取就绪 | 遍历整个 fd_set | 遍历整个 pollfd[] | 仅遍历 epoll_wait 返回的 events[] |
| 内核↔用户拷贝 | 每次调用拷贝 fd 集合 | 每次调用拷贝 pollfd[] | 仅拷贝就绪的 fd 信息 |
| 触发模式 | 水平触发 | 水平触发 | 支持 LT / ET |
| 平台 | 通用（POSIX） | 通用（POSIX） | Linux 特有；BSD/macOS 用 kqueue 思路类似 |

---

## 四、在中间件中的使用（简要）

| 技术 | 典型使用场景 |
|------|--------------|
| **零拷贝 sendfile** | Kafka 读 .log 发 Consumer、Nginx 静态文件、文件下载/流式传输 |
| **NIO 多路复用** | Kafka Broker、Redis 单线程事件循环、Nginx、Netty、各类网关与 RPC |

Kafka 流程中涉及这两部分时，可参见本文档；Kafka 自身的写入/消费流程与 acks、刷盘等仍保留在 Kafka 流程文档中。
