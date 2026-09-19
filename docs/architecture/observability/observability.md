# 可观测性（observability）域总览

> 本域包含以下叶子子系统；各叶子详情见对应文档。
> 源码基准：`github.com/cloudwego/kitex`，commit `4fffa48`。

## 1. 域职责

本域负责 RPC 可观测性：统计与 Tracer、RPC 上下文元数据、事件总线/队列、结构化日志与日志 ID、错误码与业务错误、诊断探针与性能剖析。这一层是主链路的旁路，采集统计并交给用户注入的后端。

## 2. 叶子索引

| 叶子 | 文档 | 架构图 | 数据流图 | 职责一句话 |
|------|------|--------|----------|-----------|
| stats-rpcinfo | [stats-rpcinfo.md](stats-rpcinfo/stats-rpcinfo.md) | [架构图](stats-rpcinfo/stats-rpcinfo-architecture.html) | [数据流](stats-rpcinfo/stats-rpcinfo-dataflow.html) | RPC 统计/Tracer/rpcinfo/事件钩子 |
| klog-logid-kerrors | [klog-logid-kerrors.md](klog-logid-kerrors/klog-logid-kerrors.md) | [架构图](klog-logid-kerrors/klog-logid-kerrors-architecture.html) | — | 结构化日志、日志 ID、错误码 |
| diagnosis-profiler | [diagnosis-profiler.md](diagnosis-profiler/diagnosis-profiler.md) | [架构图](diagnosis-profiler/diagnosis-profiler-architecture.html) | — | 诊断探针与性能剖析 |

## 3. 域级机制细节

- RPCStats/event 对象用 sync.Pool 复用，是高并发热路径的分配优化。
- Bus 同步分发回调（用户需自保短），Queue 缓冲后 Dump；klog 门面可 SetLogger 替换。
- 观测全程旁路，不影响主 RPC 成功路径。

## 4. 域级图（可选）

本域不单独出域级架构图，由各叶子图覆盖。
