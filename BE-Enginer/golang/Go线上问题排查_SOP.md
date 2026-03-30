# Go 服务线上问题排查 SOP：CPU 飙高 / 内存飙高

> 线上出现 **CPU 飙高** 或 **内存飙高** 时，可按本 SOP 先止血、再定位根因。与 [GMP.md](./GMP.md)、[GC.md](./GC.md)、[内存设计.md](./内存设计.md) 结合理解。

---

## 一、通用原则

1. **先止血**：必要时重启/扩容/限流，保证业务可用，再留时间做根因分析。
2. **保留现场**：在重启前尽量抓取 pprof、goroutine dump、堆信息（见下），便于事后分析。
3. **区分宿主与容器**：在 K8s/容器内看的是**容器**的 CPU/内存上限，需确认 limit 与节点资源。

---

## 二、CPU 飙高排查 SOP

### 2.1 确认是谁在吃 CPU

| 步骤 | 命令/操作 | 说明 |
|------|-----------|------|
| 1. 定位进程 | `top -H -p <pid>` 或 `ps -eLo pid,tid,pcpu,comm \| sort -k3 -rn \| head` | 看是否为本 Go 进程，以及线程级 CPU 分布（Go 里多为 M 对应线程）。 |
| 2. 看 goroutine 数量 | 若已开 pprof：`curl http://<host>:<port>/debug/pprof/goroutine?debug=2`；或 `kill -USR1 <pid>` 后看日志 | goroutine 暴增可能导致调度与执行路径变多，放大 CPU。 |
| 3. 抓 CPU profile | `go tool pprof -http=:8080 "http://<host>:<port>/debug/pprof/profile?seconds=30"` | 采 30 秒 CPU，在浏览器看火焰图或 Top，找到占用 CPU 最高的函数调用。 |
| 4. 看 goroutine 栈 | `curl "http://<host>:<port>/debug/pprof/goroutine?debug=2" > goroutine.txt`，用 grep 或肉眼找重复栈 | 大量相同栈 = 某段逻辑起过多 G 或卡在某种循环/系统调用。 |

### 2.2 常见根因与对应手段

| 可能原因 | 如何验证 | 处理方向 |
|----------|----------|----------|
| **死循环 / 密集计算** | CPU profile 火焰图里某函数占比极高 | 优化算法、加缓存、限频、异步化。 |
| **goroutine 过多且都在跑** | goroutine 数量大 + profile 里调度/运行路径多 | 限并发（worker 池、semaphore）、避免无界起 G。 |
| **系统调用 / CGO 占满** | profile 里 syscall、cgo 或某 .so 占比高 | 减少阻塞 syscall、优化 CGO 或改为纯 Go。 |
| **GC 占用高** | profile 里 `runtime.gcDrain`、`runtime.scanobject` 等占比高 | 见「内存飙高」：降分配、调 GOGC、查泄漏。 |
| **阻塞在锁/IO 导致不断重试** | goroutine 栈里大量卡在 `sync.Mutex`、channel、网络读 | 看锁竞争、慢 IO；优化逻辑或加超时/降并发。 |

### 2.3 简要流程（清单）

1. 确认是当前 Go 进程/容器 CPU 高。
2. 抓 **CPU profile（约 30s）**，用 `go tool pprof` 看火焰图/Top，找到热点函数。
3. 抓 **goroutine 栈**（debug=2），看是否有大量重复栈或异常阻塞点。
4. 结合业务（最近发布、流量、配置）判断：死循环、暴增 G、GC、锁/IO 等，再按上表处理。

---

## 三、内存飙高排查 SOP

### 3.1 确认内存用在哪

| 步骤 | 命令/操作 | 说明 |
|------|-----------|------|
| 1. 看进程/容器内存 | `top` 看 RES；容器内用 `cgroup` 或 `docker stats` / K8s 指标 | 区分是进程 RSS 高还是容器 limit 小。 |
| 2. 抓 heap profile | `go tool pprof -http=:8080 "http://<host>:<port>/debug/pprof/heap"` | 看当前堆上对象与分配来源（inuse 或 alloc）。 |
| 3. 看 goroutine 数量与栈 | 同 CPU 节：`/debug/pprof/goroutine?debug=2` | 大量 goroutine 会占栈内存（约 2KB～2MB/G，可配置）。 |
| 4. 看 GC 与分配量 | 若暴露了 `/debug/pprof/heap` 或 metrics，看 GC 次数、分配速率 | 分配过快会推高内存并加重 GC。 |
| 5. 对比两次 heap | 间隔一段时间抓两次 heap，用 `pprof -base` 做 diff | 看**增长**最多的对象与调用栈，常对应泄漏或热点路径。 |

### 3.2 常见根因与对应手段

| 可能原因 | 如何验证 | 处理方向 |
|----------|----------|----------|
| **goroutine 泄漏** | goroutine 数量只增不减；栈里大量卡在 channel 等待、select、Lock | 找到未关闭的 channel、未 Done 的 WaitGroup、死锁或永远不返回的 G，修逻辑或加超时/退出。 |
| **全局/长生命周期对象无限增长** | heap 里某类型对象或某包分配量很大；两次 heap diff 该处持续增长 | 缩小缓存、限制大 slice/map、避免全局无界队列。 |
| **大对象/大 slice 未释放** | heap 里单次分配很大（大 buffer、大数组） | 用 sync.Pool、复用 buffer、控制单次分配大小。 |
| **GC 跟不上分配** | 分配速率高，GC 频繁但堆仍涨；profile 里 GC 占比高 | 降低分配（复用、池化）、调大 GOGC 或设 GOMEMLIMIT（Go 1.19+），并排查泄漏。 |
| **CGO / 外部库持有内存** | heap 不占大头但 RSS 高；或 profile 里 C 侧分配 | 检查 CGO 与第三方库文档，避免长期持有大块内存。 |

### 3.3 简要流程（清单）

1. 确认是进程 RSS 高还是容器/节点资源不足。
2. 抓 **heap profile**，看 inuse/alloc 排名与调用栈；必要时**抓两次做 diff** 看增长。
3. 抓 **goroutine 栈**，看数量是否持续增长、是否有大量相同阻塞栈（泄漏或热点）。
4. 结合 GOGC、GOMEMLIMIT、分配速率判断：泄漏、大对象、分配过快、GC 参数等，再按上表处理。

---

## 四、常用命令与接入 pprof 的方式

### 4.1 标准库 net/http/pprof（推荐）

若服务是 `http` 包起的：

```go
import _ "net/http/pprof"
// 在已有 http 服务上无需再 Listen，/debug/pprof/ 会自动挂上
```

若非 http 服务，可单独起一个 debug 端口：

```go
go func() {
    log.Println(http.ListenAndServe(":6060", nil))
}()
```

### 4.2 常用端点与命令

| 端点 | 用途 |
|------|------|
| `/debug/pprof/profile?seconds=30` | CPU profile，采 30 秒。 |
| `/debug/pprof/heap` | 堆内存快照（inuse/alloc）。 |
| `/debug/pprof/goroutine?debug=2` | 所有 goroutine 的栈（文本）。 |
| `/debug/pprof/block` | 阻塞相关 profile。 |

本地分析示例：

```bash
# CPU 火焰图（先抓 profile）
go tool pprof -http=:8080 "http://<host>:<port>/debug/pprof/profile?seconds=30"

# 堆内存
go tool pprof -http=:8080 "http://<host>:<port>/debug/pprof/heap"

# 两次 heap 做 diff（先保存两份）
go tool pprof -http=:8080 -base=heap_old.pb.gz heap_new.pb.gz
```

### 4.3 无 pprof 时的兜底

- **GODEBUG**：`GODEBUG=schedtrace=1000` 可打印调度概况（goroutine 数等），辅助判断 G 是否暴增。
- **信号**：`kill -USR1 <pid>` 会让 Go 程序把 goroutine 栈打到 stderr（若未重定向则需从日志/终端拿）。
- **核心转储**：在可接受的前提下 `kill -SIGABRT <pid>` 或配合系统配置生成 coredump，事后用 `go tool pprof` 分析（需保留二进制）。

---

## 五、与 Go 运行时知识的衔接

- **CPU 高**：多与「谁在跑」有关——GMP 里 G 的执行、M 上的系统调用、GC 的 mark（见 [GC.md](./GC.md)）；goroutine 过多会放大调度与执行路径。
- **内存高**：与堆分配、GC 回收、goroutine 栈有关（见 [内存设计.md](./内存设计.md)、[GC.md](./GC.md)）；泄漏或分配过快会表现为 RSS 持续增长或 GC 压力大。
- **pprof 里的含义**：heap 看的是堆对象与分配栈；goroutine 栈看的是当前所有 G 的调用链，便于发现泄漏与阻塞点。

---

## 六、SOP 速查表

| 现象 | 优先动作 | 关键数据 |
|------|----------|----------|
| **CPU 飙高** | 抓 30s CPU profile → 火焰图/Top 找热点；抓 goroutine 栈看是否暴增/重复栈 | profile 热点函数、goroutine 数量与栈 |
| **内存飙高** | 抓 heap profile；抓 goroutine 数量与栈；必要时两次 heap 做 diff | heap inuse/alloc、goroutine 数、diff 增长对象 |
| **两者都高** | 先看 goroutine 是否暴增（占栈+调度开销）；再分别看 CPU 热点与 heap/GC | goroutine 栈、CPU profile、heap、GC 指标 |

**事前建议**：线上服务预留 `/debug/pprof`（可加鉴权或仅内网），并约定在重启前执行「抓 profile + goroutine 栈」的 checklist，便于事后分析。
