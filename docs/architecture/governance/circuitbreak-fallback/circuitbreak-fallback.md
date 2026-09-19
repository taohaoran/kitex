# 熔断与降级（circuitbreak-fallback）

> 本文是 `governance` 域下的叶子子系统文档。域级总览见 `../governance.md`，本文只展开 `pkg/circuitbreak/`（熔断器）与 `pkg/fallback/`（降级 fallback），不展开重试如何调用熔断器（见 `../retry/retry.md`）。
>
> 源码基准：`github.com/cloudwego/kitex`，commit `4fffa48`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| 熔断中间件 | `NewCircuitBreakerMW(Control, Panel)`：查 IsAllowed→放行/拒绝→RecordStat | `pkg/circuitbreak/circuitbreak.go:80` |
| 流式熔断中间件 | `NewStreamCircuitBreakerMW`：streaming 版本 | `circuitbreak.go:100` |
| 熔断控制策略 | `Control`：GetKey/GetErrorType/DecorateError | `circuitbreak.go:67` |
| 结果统计上报 | `RecordStat`：按 ErrorType 调 panel.Timeout/Fail/Succeed | `circuitbreak.go:120` |
| 错误类型 | ErrorType：Ignorable/Timeout/Failure/Success | `circuitbreak.go:45` |
| 错误包装 | `WrapErrorWithType`：给业务错误标注熔断可见性 | `circuitbreak.go:60` |
| 熔断套件 | `CBSuite`：持有 service 级 + instance 级双 Panel/Control | `pkg/circuitbreak/cbsuite.go:94` |
| 两级中间件 | `ServiceCBMW()` / `InstanceCBMW()` | `cbsuite.go:132,150` |
| 动态配置 | `UpdateServiceCBConfig/UpdateInstanceCBConfig` | `cbsuite.go:176,182` |
| 熔断参数 | `Parameter`：Enabled/ErrorRate/MinimalSample；`CBConfig` | `circuitbreak.go:32`、`cbsuite.go:52` |
| 降级策略 | `fallback.Policy` + `ErrorFallback/TimeoutAndCBFallback/NewFallbackPolicy` | `pkg/fallback/fallback.go:32,43,56,65` |
| 降级执行 | `Policy.DoIfNeeded`：失败时执行用户降级函数 | `fallback.go:106` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|-------------|------|------|
| `Control` | `circuitbreak.go:67` | 熔断控制面：如何分 key、如何判错误类型、如何装饰熔断错误 |
| `Parameter`/`CBConfig` | `circuitbreak.go:32`、`cbsuite.go:52` | 熔断阈值（错误率、最小样本） |
| `CBSuite` | `cbsuite.go:94` | 一个 client 的熔断套件；servicePanel 按服务聚合、instancePanel 按实例聚合 |
| `circuitbreaker.Panel` | 外部（bytedance/gopkg） | 熔断状态机面板：IsAllowed/Timeout/Fail/Succeed |
| `Policy` | `fallback.go:65` | 降级策略：持有 fallbackFunc，DoIfNeeded 判定是否执行 |
| `Func` / `RealReqRespFunc` | `fallback.go:99,103` | 降级函数签名 |

## 3. 关键调用链

1. **熔断中间件放行/拒绝**：`NewCircuitBreakerMW` 返回的 endpoint 中间件（`circuitbreak.go:81-96`）先 `control.GetKey(ctx, request)` 得 key；未启用直接透传；`!panel.IsAllowed(key)` 即熔断打开，返回 `DecorateError(...kerrors.ErrCircuitBreak)`（`circuitbreak.go:88-89`）；否则调 `next(ctx, request, response)`，返回后 `RecordStat`（`circuitbreak.go:92-93`）。
2. **统计驱动状态迁移**：`RecordStat`（`circuitbreak.go:120`）按 `GetErrorType` 结果分别调 `panel.Timeout/Fail/Succeed`；Panel 内部累计窗口错误率，超 `ErrorRate` 且样本≥`MinimalSample` 则 Open，冷却后 HalfOpen 试探。
3. **降级执行**：RPC 返回 err 后 `fallback.Policy.DoIfNeeded`（`fallback.go:106`）取出业务错误，调用 `fallbackFunc`（`fallback.go:119`）执行用户降级；若降级既无响应又无错误，回退返回原始错误（`fallback.go:121-124`）。

## 4. 配置项

| flag / option | 默认 / 行为 | 位置 |
|---------------|-------------|------|
| `CBConfig.ErrorRate` | 触发熔断的错误率阈值 | `cbsuite.go:52` |
| `CBConfig.MinimalSample` | 触发熔断所需最小样本数 | `circuitbreak.go:38` |
| `CBSuiteOption` | 自定义 service/instance 级 GetErrorTypeFunc | `cbsuite.go:120`、`cbsuite_option.go` |
| `ErrorFallback` / `TimeoutAndCBFallback` | 分别在"出错"或"超时+熔断"时触发降级 | `fallback.go:32,43` |
| `EnableReportAsFallback` | 降级结果是否上报为 fallback 指标 | `fallback.go:70` |

## 5. 错误与重试语义

- 熔断打开时中间件直接返回 `kerrors.ErrCircuitBreak`（经 DecorateError 装饰），不再调用下游；该错误会被 retry 识别为可停止重试。
- `TypeIgnorable` 错误不计入熔断（`circuitbreak.go:47`）。
- 降级函数返回的错误替代原始错误；降级本身失败不二次重试。
- 与 retry 的关系：retry 在熔断打开时 `circuitBreakerStop` 立即停止（见 retry 叶子），二者协同。

## 6. 并发细节

- 熔断 Panel 是并发安全的（由外部 gopkg 实现）；Kitex 这层只做 GetKey/IsAllowed/RecordStat 调用，无锁。
- `CBSuite` 的 servicePanel/instancePanel 在建时初始化，运行中只读；`UpdateServiceCBConfig` 热更新配置（`cbsuite.go:176`）。
- 每 client 一个 CBSuite（注释强调不共享 event.Queue，`cbsuite.go:111`）。
- context：GetKey/GetErrorType 接收 ctx，支持按请求元数据分 key。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `pkg/circuitbreak/`：Control/CBSuite/中间件/RecordStat。
- `pkg/fallback/`：Policy 与降级函数执行。

**Out-of-Scope（不在本仓库源码内）**
- `github.com/bytedance/gopkg/cloud/circuitbreaker`：Panel 状态机实现（Closed/Open/HalfOpen、滑动窗口、冷却计时）——不在本仓库。
- 重试与熔断的协同见 `../retry/retry.md`；指标上报见 `../stats-rpcinfo/stats-rpcinfo.md`。

## 8. 与相邻子系统交互

- 上游 → 本叶子：client/server 把 `ServiceCBMW/InstanceCBMW` 作为 endpoint 中间件挂入调用链。
- 本叶子 → 下游：调用 `panel.IsAllowed/RecordStat`；熔断/失败时执行 fallback 函数；错误经 `kerrors` 包装向上传播。
- 流向：请求 → 熔断中间件（查 Panel）→ 下游 RPC → RecordStat → 失败则 fallback。

## 9. 语言专项适配口径

- **并发模型**：无锁包装层；真正并发安全的状态机在外部 Panel。无 goroutine、无 channel。
- **控制器模式**：状态机由外部 Panel 维护（Closed→Open→HalfOpen→Closed），本叶子只是调用方；lifecycle 图表达的是 Panel 状态迁移，非 Kitex 自实现。
- **多二进制与部署边界**：纯运行时库。
- **internal 边界与依赖方向**：`pkg/circuitbreak` 依赖 `pkg/endpoint`、`pkg/kerrors`、`pkg/event`；被 client/server 治理中间件与 retry 依赖，方向单向。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|----------------|
| 熔断与降级架构图 | `circuitbreak-fallback-architecture.html` | architecture | showcase |
| 熔断器状态机 | `circuitbreak-fallback-lifecycle.html` | lifecycle | standard（lifecycle 对双向边标签间距严格，labelDy 错位后通过标准档；showcase 未过，按流程降档，render 退出码 0） |

- 本叶子不补 sequence/dataflow：放行/拒绝是同步中间件调用（已在架构图表达）；状态机由 lifecycle 表达。
- JSON IR 源文件：`json/circuitbreak-fallback-architecture.json`、`json/circuitbreak-fallback-lifecycle.json`。
