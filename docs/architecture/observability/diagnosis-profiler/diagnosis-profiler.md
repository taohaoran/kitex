# 诊断与性能剖析（diagnosis-profiler）

> 本文是 `observability` 域下的叶子子系统文档。域级总览见 `../observability.md`，本文只展开 `pkg/diagnosis/`（诊断探针注册）与 `pkg/profiler/`（性能剖析采样），不展开常规 metrics（见 `../stats-rpcinfo/stats-rpcinfo.md`）。
>
> 源码基准：`github.com/cloudwego/kitex`，commit `4fffa48`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| 诊断服务接口 | `diagnosis.Service.RegisterProbeFunc` | `pkg/diagnosis/interface.go:27` |
| 探针函数 | `ProbeFunc func() interface{}` + `RegisterProbeFunc` | `interface.go:24,35` |
| 探针包装 | `WrapAsProbeFunc(data)` | `interface.go:58` |
| 空实现 | `noopService`（无调试服务时兜底） | `interface.go:68` |
| 剖析器接口 | `profiler.Profiler`：Prepare/State/Stop | `pkg/profiler/profiler.go:37` |
| 采样标签 | `Tag/Untag/IsEnabled(ctx)` | `profiler.go:103,111,117` |
| 处理器 | `Processor` + `LogProcessor` | `profiler.go:48,50` |
| 周期聚合 | `NewProfiler(processor, interval, window)` | `profiler.go:66` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|-------------|------|------|
| `diagnosis.Service` | `interface.go:27` | 调试服务注册探针 |
| `ProbeFunc` | `interface.go:24` | 无参返回 interface{} 的诊断 dump 函数 |
| `profiler.Profiler` | `profiler.go:37` | 剖析器抽象 |
| `Processor` | `profiler.go:48` | 接收聚合 profile 的回调 |
| `profilerContext` | `profiler.go:123` | ctx 上的采样标记 |

## 3. 关键调用链

1. **探针注册**：组件初始化时 `diagnosis.RegisterProbeFunc(svc, name, probeFunc)`（`interface.go:35`）把 dump 函数挂到调试服务；调试 HTTP 端点按 name 拉取。
2. **剖析采样**：请求进入时 `Profiler.Prepare(ctx)`（`profiler.go:137`）返回带标记的 ctx；`IsEnabled(ctx)`（`profiler.go:111`）判定当前是否采样；周期到了把窗口内聚合的 `TagsProfile` 交 `Processor`（`profiler.go:48`）。
3. **停止**：`Stop()`（`profiler.go:153`）收尾。

## 4. 配置项

| flag / option | 默认 / 行为 | 位置 |
|---------------|-------------|------|
| `NewProfiler(processor, interval, window)` | 采样周期与聚合窗口 | `profiler.go:66` |

## 5. 错误与重试语义

- 诊断/剖析是旁路，失败不影响主链路。
- `IsEnabled` 内联返回 false 快速路径（注释强调，`profiler.go:110-111`）。

## 6. 并发细节

- profiler 周期聚合用后台 goroutine（NewProfiler 起），Stop 收尾。
- ctx 上用 `profilerContextKey`（`profiler.go:35`）做采样标记，无共享锁。
- diagnosis 注册在初始化期完成，运行期只读。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `pkg/diagnosis/`、`pkg/profiler/`。

**Out-of-Scope（不在本仓库源码内）**
- 调试 HTTP 服务端如何暴露探针，由 kitex server 集成层提供；pprof 等底层剖析由 Go runtime 提供。
- 上报后端见 `../stats-rpcinfo/stats-rpcinfo.md`。

## 8. 与相邻子系统交互

- 上游 → 本叶子：server 集成层构造 diagnosis.Service；请求路径调 profiler.Prepare/IsEnabled。
- 本叶子 → 下游：Processor 把 profile 交给用户提供的上报函数。
- 流向：组件注册探针 → 调试端拉取；请求打标 → 周期聚合 → Processor 上报。

## 9. 语言专项适配口径

- **并发模型**：profiler 后台周期 goroutine；ctx 采样标记无锁。
- **控制器模式**：非 Reconcile。
- **多二进制与部署边界**：纯运行时库。
- **internal 边界与依赖方向**：两包均为薄接口/工具，被 server 集成层依赖，方向向下。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|----------------|
| 诊断与剖析架构图 | `diagnosis-profiler-architecture.html` | architecture | showcase |

- 本叶子不补 sequence/dataflow：两包极薄，注册与周期聚合已在 MD 文字说明。
- JSON IR 源文件：`json/diagnosis-profiler-architecture.json`。
