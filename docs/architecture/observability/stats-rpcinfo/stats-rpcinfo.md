# RPC 统计 / 上下文 / 事件钩子（stats-rpcinfo）

> 本文是 `observability` 域下的叶子子系统文档。域级总览见 `../observability.md`，本文只展开 `pkg/stats/`（RPC 统计与 Tracer）、`pkg/rpcinfo/`（RPC 上下文元数据）、`pkg/event/`（事件总线与队列），不展开日志（见 `../klog-logid-kerrors/klog-logid-kerrors.md`）与诊断（见 `../diagnosis-profiler/diagnosis-profiler.md`）。
>
> 源码基准：`github.com/cloudwego/kitex`，commit `4fffa48`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| 追踪器接口 | `stats.Tracer.Start/Finish` | `pkg/stats/tracer.go:24` |
| 事件定义 | `Event/Level/EventIndex`，预定义 RPC 阶段事件 | `pkg/stats/event.go:26,39,136` |
| 统计状态 | `stats.Status`（OK/Error） | `pkg/stats/status.go:20` |
| RPC 上下文 | `RPCInfo`：From/To/Invocation/RPCConfig | `pkg/rpcinfo/interface.go:105` |
| 调用信息 | `Invocation`：方法/Extra/biz 状态 | `rpcinfo/interface.go:93` |
| RPC 统计 | `RPCStats`：耗时/错误/panic | `rpcinfo/interface.go:41`、`rpcstats.go` |
| 对象复用 | `rpcStatsPool/eventPool` sync.Pool | `rpcinfo/rpcstats.go:35-36` |
| 事件总线 | `event.Bus.Watch/Dispatch/DispatchAndWait` | `pkg/event/bus.go:31,48,73` |
| 事件队列 | `event.Queue.Push/Dump` | `pkg/event/queue.go:55,76` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|-------------|------|------|
| `stats.Tracer` | `tracer.go:24` | RPC 起止钩子；Start 返回新 ctx 可携带追踪态 |
| `stats.Event` | `event.go:39` | 事件索引+级别；DefineNewEvent 扩展 |
| `rpcinfo.RPCInfo` | `interface.go:105` | 一次 RPC 的完整元数据上下文 |
| `RPCStats` | `interface.go:41` | 统计字段集合 |
| `event.Bus` | `bus.go:31` | 按事件名注册回调，Dispatch 分发 |
| `event.Queue` | `queue.go:55` | 缓冲事件供 Dump |

## 3. 关键调用链

1. **RPC 起止追踪**：client/server 在 RPC 开始调 `tracer.Start(ctx)`（`tracer.go:25`）返回带追踪态的 ctx；结束调 `tracer.Finish(ctx)`（`tracer.go:26`），此时从 RPCStats 读取统计并上报。
2. **事件分发**：`bus.Watch(event, callback)`（`bus.go:48`）注册；运行中 `bus.Dispatch(event)`（`bus.go:73`）同步调用各 callback；`DispatchAndWait`（`bus.go:85`）等待全部完成。
3. **对象复用**：每次 RPC 用 `rpcStatsPool.Get()`（`rpcstats.go:35`）取 RPCStats，结束 `Recycle`（`rpcstats.go:84`）回池，避免热路径分配。

## 4. 配置项

| flag / option | 默认 / 行为 | 位置 |
|---------------|-------------|------|
| `SetDefaultEventNum` | 事件队列默认容量 | `event/queue.go:40` |
| `MaxEventNum/PredefinedEventNum` | 事件总数与预定义数 | `stats/event.go:152,159` |
| `DefineNewEvent` | 用户自定义事件名 | `stats/event.go:136` |

## 5. 错误与重试语义

- Tracer 上报失败不影响主 RPC 路径（观测旁路）。
- `RPCStats.Panicked()`（`interface.go:50`）记录 RPC 中的 panic 供上报。
- 无重试；观测是旁路副作用。

## 6. 并发细节

- **sync.Pool**：RPCStats 与 event 对象用 Pool 复用（`rpcstats.go:35-36`），是高并发下的分配优化。
- **Bus 回调**：Dispatch 同步执行回调；用户回调应短，否则阻塞 RPC。
- **Queue**：缓冲队列，Push 非阻塞或有界。
- context：RPCInfo 挂在 ctx 上随调用链传递；Start 返回新 ctx 携带追踪态。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `pkg/stats/`、`pkg/rpcinfo/`、`pkg/event/`。

**Out-of-Scope（不在本仓库源码内）**
- 具体 metrics/tracing 后端（Prometheus、OpenTelemetry 等）不在本仓库，由用户注入 Tracer/Bus 回调。
- 结构化日志见 `../klog-logid-kerrors/klog-logid-kerrors.md`。

## 8. 与相邻子系统交互

- 上游 → 本叶子：client/server 调用链在起止点调用 Tracer；各中间件读 RPCInfo。
- 本叶子 → 下游：Bus 回调把事件交给用户注册的监控/追踪后端。
- 流向：RPC 开始 → Start 取 RPCInfo → 调用 → Finish 读 RPCStats → Bus 分发 → 后端。

## 9. 语言专项适配口径

- **并发模型**：sync.Pool 对象复用是 Kitex 热路径核心优化；Bus 同步回调模型（用户需自保短）。
- **控制器模式**：非 Reconcile；这是请求旁路的观测钩子。
- **多二进制与部署边界**：纯运行时库。
- **internal 边界与依赖方向**：`pkg/rpcinfo` 是被广泛依赖的基础包；`pkg/event` 被限流/熔断/circuitbreak 等用于事件上报。依赖方向以 rpcinfo 为底层，上层依赖它。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|----------------|
| 统计与事件架构图 | `stats-rpcinfo-architecture.html` | architecture | showcase |
| 事件采集数据流 | `stats-rpcinfo-dataflow.html` | dataflow | showcase |

- 本叶子不补 sequence：事件分发是同步回调，已由数据流表达。
- JSON IR 源文件：`json/stats-rpcinfo-architecture.json`、`json/stats-rpcinfo-dataflow.json`。
