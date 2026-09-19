# 客户端中间件链与超时（client-middleware-endpoint）

> 本文是 `client-core` 域下的叶子子系统文档。域级总览见 `../client-core.md`。
> 本文只展开「客户端中间件链编织、rpcinfo/context 透传、RPCTimeout 中间件与 worker pool」，不展开 NewClient 入口与选项装载（见 `../client-entry/client-entry.md`），也不展开负载均衡器/服务发现的具体算法实现（`pkg/loadbalance`、`pkg/discovery` 归其他分片）。
>
> 源码基准：`github.com/cloudwego/kitex`，Go 1.20，commit `4fffa48`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| 中间件链编织入口 `initMiddlewares` | 按固定顺序装配通用/一元/流三类中间件切片，供 `buildInvokeChain` 用 `endpoint.Chain` 串成调用链 | `client/client.go:297 (*kClient).initMiddlewares` |
| 服务发现中间件 `newResolveMWBuilder` | 从 RPCInfo 取目标，经 `lbf.Get` 拿负载均衡器，`picker.Next` 选实例；可重试错误最多 `maxRetry=6` 次重选 | `client/middlewares.go:113 newResolveMWBuilder` |
| 代理中间件 `newProxyMW` | 若 `proxy.ForwardProxy` 实现 `proxy.WithMiddleware` 则用其自定义中间件，否则默认 `ResolveProxyInstance` 后透传 | `client/middlewares.go:43 newProxyMW` |
| IO 错误处理中间件 `newIOErrorHandleMW` | 包装下游错误；默认 `DefaultClientErrorHandler` 把远端错误包成 `kerrors.ErrRemoteOrNetwork` | `client/middlewares.go:176 newIOErrorHandleMW` |
| 服务发现事件分发 `discoveryEventHandler` | 把 resolver 的 `discovery.Change` 同时推到 event bus（同步 dispatch）与 event queue（快照化 extra） | `client/middlewares.go:88 discoveryEventHandler` |
| 单次调用选项 ctx 透传 `NewCtxWithCallOptions` / `CallOptionsFromCtx` | 把 `[]callopt.Option` 挂到 ctx，`applyCallOptions` 时取出 | `client/context.go:33`、`:41` |
| 上下文级中间件 `WithContextMiddlewares` / `contextMW` | 把一组中间件挂到 ctx，`contextMW` 在链中执行；优先级高于客户端级中间件 | `client/context_middleware.go:30`、`:48` |
| RPCTimeout 一元中间件 `rpcTimeoutMW` | 读 RPCInfo 的 RPCTimeout，经 `workerPool` 跑下游 endpoint，超时返回 `ErrRPCTimeout` 并区分业务超时/业务取消 | `client/rpctimeout.go:84 rpcTimeoutMW` |
| 超时错误构造 `makeTimeoutErr` | 区分 `context.Canceled`（业务取消）、业务超时（`ErrTimeoutByBusiness`）、kitex 超时（`ErrRPCTimeout`） | `client/rpctimeout.go:36 makeTimeoutErr` |
| 超时 worker pool `timeoutPool` | 复用 goroutine 执行带超时的 endpoint，减少每请求起 goroutine 开销；空闲 worker 自动退出 | `client/rpctimeout_pool.go:33 timeoutPool`、`:137 RunTask` |
| 超时任务 `timeoutTask` | 在 worker goroutine 跑 `ep`，`Wait` 用 timer 等待完成/父 ctx 取消/超时；`sync.Pool` 复用 | `client/rpctimeout_pool.go:161 timeoutTask`、`:204 Run`、`:232 Wait` |
| 超时上下文 `timeoutContext` | 自定义 `context.Context`，合并父 ctx deadline 与本次 timeout，`Cancel` 关闭 done channel | `client/rpctimeout_pool.go:276 timeoutContext` |
| endpoint 中间件原语 `endpoint.Chain` / `UnaryChain` / `cep.StreamChain` | 纯函数式中间件编织：从后往前逐层包装 `next` | `pkg/endpoint/endpoint.go:31`、`pkg/endpoint/unary_endpoint.go:27`、`pkg/endpoint/cep/endpoint.go:63` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|-------------|------|------|
| `endpoint.Endpoint` | `pkg/endpoint/endpoint.go:22` | `func(ctx, req, resp) error`，调用链最内层函数类型 |
| `endpoint.Middleware` | `pkg/endpoint/endpoint.go:25` | `func(next Endpoint) Endpoint`，装饰器 |
| `endpoint.MiddlewareBuilder` | `pkg/endpoint/endpoint.go:28` | `func(ctx) Middleware`，可从构造期 ctx 取 bus/queue |
| `endpoint.Chain` | `pkg/endpoint/endpoint.go:31` | 把多个 Middleware 串成一个；`CtxEventBusKey`/`CtxEventQueueKey` 常量供 builder 取事件总线 |
| `endpoint.UnaryEndpoint` / `UnaryMiddleware` / `UnaryChain` | `pkg/endpoint/unary_endpoint.go:21/23/27` | 一元专用 endpoint 类型，与通用 Endpoint 可互转（`ToUnaryMiddleware`） |
| `cep.StreamEndpoint` / `StreamMiddleware` / `StreamChain` | `pkg/endpoint/cep/endpoint.go:28/31/63` | 流专用：`func(ctx) (ClientStream, error)`；另有 `StreamRecvChain`/`StreamSendChain` 包装收发 |
| `middleware`（client 内部） | `client/client.go:278` | 汇总 `mws/uMws/smws/sMws` 四类切片的容器 |
| `timeoutPool` | `client/rpctimeout_pool.go:33` | `tasks chan *timeoutTask` + 原子 `size` + 空闲 ticker；`maxIdle=128`、`maxIdleTime=1min` |
| `timeoutTask` | `client/rpctimeout_pool.go:161` | 持有 `*timeoutContext`、`req/resp`、`ep`、`err atomic.Value`、`wg`；`sync.Pool` 复用 |
| `timeoutContext` | `client/rpctimeout_pool.go:276` | 内嵌父 ctx，自定义 `dl/ch/mu/err`；`Cancel` 幂等关闭 done |
| `instSnapshot` / `discoveryEventExtra` | `client/middlewares.go:63/69` | 事件 extra 的值类型快照，避免持有大 tags map；实现 `KitexDumpLazyExtra` |

## 3. 关键调用链

### 链路一：客户端中间件链的装配与执行顺序

1. `initMiddlewares(ctx)`（`client/client.go:297`）按注释固定顺序装配：
   - 通用 `mws`：`contextMW` → 用户 `MWBs` → `acl.NewACLMiddleware` →（无 proxy 时）`newResolveMWBuilder(lbf)` → `CBSuite.InstanceCBMW()` → 用户 `IMWBs`；有 proxy 时按是否有 resolver 决定是否插 resolve，再 `newProxyMW`；最后 `newIOErrorHandleMW`。
   - 一元 `uMws`：`CBSuite.ServiceCBMW().ToUnaryMiddleware()` →（xDS 开启时）`XDSRouterMiddleware` → `rpcTimeoutMW(ctx)` → 用户 `UnaryOptions.UnaryMiddlewares`。
   - 流 `smws`/`sMws`：`smws = mws` 前置 xDS router；`sMws = [CBSuite.StreamingServiceCBMW(), ...StreamMiddlewares]`。
2. `buildInvokeChain`（`client/client.go:479`）：`eps = endpoint.Chain(mw.mws...)(innerHandlerEp)`，再 `kc.eps = endpoint.UnaryChain(mw.uMws...)(func{ eps(...) })`。即一元链 = uMws 包在 mws 外层。
3. 运行时 `Call` 调 `kc.eps(ctx, req, resp)`（`client/client.go:410`），从最外层 uMws 依次进入 mws，最终到 `invokeHandleEndpoint`。

### 链路二：服务发现中间件的选实例与重试

1. `newResolveMWBuilder(lbf)(ctx)`（`client/middlewares.go:113`）返回中间件。进入后先 `rpcinfo.GetRPCInfo(ctx)` 取 `To()`，若 `remote.GetInstance() != nil`（如 `callopt.WithHostPort` 已指定）直接 `next`。
2. 否则 `lbf.Get(ctx, dest)` 拿负载均衡器（`client/middlewares.go:132`）。
3. `for i := 0; i < maxRetry(=6); i++`（`client/middlewares.go:138`）：每轮先检查 `ctx.Done()`（超时则 `ErrRPCTimeout`），再 `lb.GetPicker()` 拿新 picker、`picker.Next(ctx, request)` 选实例；选中则 `remote.SetInstance(ins)` 并 `next(...)`。
4. 成功返回；`retryable(err)`（`client/middlewares.go:272`，仅 `ErrGetConnection`/`ErrCircuitBreak`）为真则记录 lastErr 继续重选；不可重试直接返回。picker 若实现 `internal.Reusable` 则 `Recycle`。

### 链路三：RPCTimeout 中间件的超时控制

1. `rpcTimeoutMW(mwCtx)`（`client/rpctimeout.go:84`）从 ctx 读 `rpctimeout.TimeoutAdjustKey` 拿到额外超时 `moreTimeout`，返回 UnaryMiddleware。
2. 进入时读 `ri.Config().RPCTimeout()`，加 `moreTimeout`；若 `ctx.Done()==nil && tm<=0` 走 fast path 直接 `next`（`client/rpctimeout.go:119`）。
3. 否则 `start := time.Now()`，`workerPool.RunTask(ctx, tm, req, resp, backgroundEP)`（`client/rpctimeout.go:123`）。`RunTask`（`client/rpctimeout_pool.go:137`）把 `timeoutTask` 投入 `tasks` channel；有空闲 worker 则复用，否则 `createWorker`（`client/rpctimeout_pool.go:97`）新开 goroutine 跑 `t.Run()`，满了就 `go t.Run()` 兜底。
4. `t.Run()`（`client/rpctimeout_pool.go:204`）在 worker 里执行 `backgroundEP(ctx, req, resp)`，defer 中 `recover` panic、`t.Cancel(context.Canceled)`、`recycle`。
5. `t.Wait()`（`client/rpctimeout_pool.go:232`）用 `time.Until(deadline)` 起 timer，select 等 `ctx.Done()`（正常完成）/父 ctx 取消/timer 超时。超时或父取消则 `t.Cancel(...)` 并返回 `ctx.Err()`。
6. 上层 `rpcTimeoutMW` 拿到非 nil err 且等于 `ctx.Err()` 时调 `makeTimeoutErr`（`client/rpctimeout.go:36`）细化错误码。

## 4. 配置项

| option / 配置 | 默认 / 行为 | 位置 |
|----------------|-------------|------|
| `client.WithMiddleware` / `WithMiddlewareBuilder` / `WithInstanceMW` | 把用户中间件追加到 `MWBs`/`IMWBs`；`IMWBs` 在 resolve+实例选择之后执行 | `client/option.go:102/113/121` |
| `client.WithUnaryMiddleware` / `WithUnaryMiddlewareBuilder` | 一元专用中间件，排在 rpctimeout 之后 | `client/option_unary.go:52/61` |
| `client.WithStreamMiddleware` 系列 | 流中间件，经 `cep.Stream*Chain` 编织 | `client/option_stream.go:81` 起 |
| `WithContextMiddlewares(ctx, mws...)` | 调用级（ctx 级）中间件，优先级高于客户端级；`contextMW` 在链中执行 | `client/context_middleware.go:30` |
| `callopt.WithRPCTimeout` / `WithConnectTimeout` | 单次调用覆盖超时并置 Locks 位；需 `client.WithRPCTimeout`/`WithTimeoutProvider` 配合 | `client/callopt/options.go:148/162` |
| `callopt.WithHostPort` / `WithTag` | 单次指定实例/标签，跳过 resolve 中间件的 picker | `client/callopt/options.go:95/172` |
| RPCTimeout worker pool | 包级 `workerPool = newTimeoutPool(128, 1min)`；未启用超时（tm<=0 且无 ctx.Done）零开销 | `client/rpctimeout.go:34` |
| `rpctimeout.LoadGlobalNeedFineGrainedErrCode` / `LoadBusinessTimeoutThreshold` | 全局开关，决定是否返回 `ErrCanceledByBusiness`/`ErrTimeoutByBusiness` 细粒度错误码与业务超时阈值 | `client/rpctimeout.go:47/61` |
| `maxRetry` | resolve 中间件选实例最大重试次数 6 | `client/middlewares.go:41` |

## 5. 错误与重试语义

- **选实例重试**：`newResolveMWBuilder` 内 `maxRetry=6` 次循环，仅对 `ErrGetConnection`/`ErrCircuitBreak`（`retryable`，`client/middlewares.go:272`）重选实例；每轮重新 `GetPicker()` 避免拿到含过期实例的旧 picker。这是「选实例级」重试，与 `RetryContainer` 的「RPC 级」重试（client-entry 叶子）是两层。
- **超时错误**：`makeTimeoutErr`（`client/rpctimeout.go:36`）分级：父 ctx `Canceled` → `ErrCanceledByBusiness`（细粒度开启时）；实际 deadline 早于 kitex 超时且阈值内 → `ErrTimeoutByBusiness`；否则 `ErrRPCTimeout`。错误信息带 `to/method/remote` 便于定位。
- **超时后残留错误**：`backgroundEP` 包装下游，若超时后下游才返回非超时错误，会 `klog.CtxErrorf` 打印（`client/rpctimeout.go:93-109`），避免该错误被静默丢弃。
- **IO 错误包装**：`DefaultClientErrorHandler`（`client/middlewares.go:210`）对 `*remote.TransError`/`protobuf.PBError`/`ApplicationException` 类错误加 `remote` 前缀包成 `ErrRemoteOrNetwork`；`ClientErrorHandlerWithAddr` 额外带远端地址。
- **panic**：`timeoutTask.Run` 内 `recover` 把 panic 转 `rpcinfo.ClientPanicToErr`（`client/rpctimeout_pool.go:206`）；`Call` 层另有兜底（client-entry 叶子）。
- **picker 回收**：picker 实现 `internal.Reusable` 时每轮 `Recycle`，避免选实例器泄漏。

## 6. 并发细节

- **goroutine 边界**：`timeoutPool` 是本叶子唯一的 goroutine 管理点。worker 由 `createWorker`（`client/rpctimeout_pool.go:97`）创建，上限 `maxIdle=128`；`createTicker`（`:61`）每 `maxIdleTime/maxIdle/10`（≥10ms）发 noop task 探测空闲，空闲超 `maxIdleTime=1min` 的 worker 退出。满员时 `RunTask` 兜底 `go t.Run()`。
- **channel 通信**：`tasks chan *timeoutTask` 是 worker 任务队列；`RunTask` 先非阻塞 `select p.tasks <- t`，失败再建 worker。`timeoutTask.wg sync.WaitGroup` 保证 `Wait` 完成后才 `recycle` 回池。
- **共享状态**：`timeoutTask.err atomic.Value` 跨 worker goroutine 与 Wait goroutine 安全传递结果；`timeoutContext.mu sync.Mutex` 保护 `err` 与 `ch` 关闭；`timeoutPool.size atomic.Int32` 统计活跃 worker。
- **对象池**：`poolTask sync.Pool`（`client/rpctimeout_pool.go:154`）复用 `timeoutTask`；但 `timeoutContext` 不复用（注释：用户可能持有引用）。
- **context 超时传播**：`timeoutContext` 合并父 ctx deadline 与本次 timeout，取更早者（`client/rpctimeout_pool.go:286`）；`Wait` 同时监听 `t.ctx.Done()`、父 `Context.Done()`、timer，任一触发即 `Cancel`。父 ctx 取消会传播到下游 endpoint。
- **事件总线**：`initContext`（`client/client.go:172`）把 `Bus`/`Events` 挂到构造期 ctx，`MiddlewareBuilder` 经 `CtxEventBusKey`/`CtxEventQueueKey` 读取；`discoveryEventHandler` 同步 dispatch 到 bus、快照化 push 到 queue。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `client/middlewares.go`：resolve/proxy/io-error 中间件与事件分发。
- `client/context.go`、`client/context_middleware.go`：callopt 与 context 级中间件的 ctx 透传。
- `client/rpctimeout.go`、`client/rpctimeout_pool.go`：RPCTimeout 中间件与 worker pool。
- `pkg/endpoint/`、`pkg/endpoint/cep/`：Endpoint/Middleware/Chain 纯函数原语与流中间件类型。

**Out-of-Scope（不在本仓库源码内）**
- 负载均衡算法与实例权重选择：`pkg/loadbalance`、`pkg/loadbalance/lbcache`（`lbf.Get`/`picker.Next` 的具体算法归其他分片）。
- 服务发现 resolver 实现：`pkg/discovery`、注册中心（外部）。
- 熔断器/限流/重试容器/ACL 的具体策略引擎：`pkg/circuitbreak`、`pkg/limit`、`pkg/retry`、`pkg/acl`（本叶子只持有中间件接入点）。
- xDS router 中间件具体实现：`pkg/xds`。
- netpoll、对端服务 —— 不在本仓库源码内。

## 8. 与相邻子系统交互

- 上游 → 本叶子：`client-entry` 叶子的 `buildInvokeChain` 调用 `initMiddlewares` 产出中间件切片；运行时 `Call` 把请求交给编织好的 `kc.eps`。
- 本叶子 → 下游：resolve 中间件调 `lbf.Get`（负载均衡，相邻分片）→ `invokeHandleEndpoint`（client-entry 叶子）→ 传输层；rpctimeout 中间件包在一元链最外层，控制整体超时。
- 横向：`pkg/endpoint` 是纯接口层，client/server 两侧共用 `Chain`；server 侧中间件链见 `server-core` 域。
- 事件侧：`discoveryEventHandler` 把实例变更推到 `event.Bus`/`event.Queue`，供诊断与连接池清理消费（`initConnPool` 监听 `ChangeEventName` 清理长连接）。

## 9. 语言专项适配口径（Go）

- **并发模型**：本叶子是「函数式中间件装饰 + worker pool 超时」模型，非 K8s Reconcile。`endpoint.Chain` 是纯函数闭包嵌套，无 goroutine；唯一并发点是 RPCTimeout 的 `timeoutPool`——用 worker goroutine 跑下游 endpoint，主 goroutine 在 `Wait` 中用 timer select 等待，是典型的「用池化 goroutine + channel + timer」做超时取消传播。
- **goroutine 生命周期**：worker 上限 128、空闲 1min 自动退出，由 ticker 周期发 noop 探测；`RunTask` 满员时 `go t.Run()` 兜底，避免阻塞调用方。`timeoutTask` 用 `sync.WaitGroup` 保证回收发生在 `Wait` 之后，杜绝 use-after-pool。
- **context 传播**：`timeoutContext` 是自定义 `context.Context` 实现，合并父 deadline 与新 timeout；`Cancel` 幂等。这是 kitex 把「RPC 级超时」叠加到「业务 ctx 超时」上的关键设施。
- **internal 边界**：`pkg/endpoint` 对外暴露纯接口与 `Chain` 原语，无内部依赖；客户端中间件实现在 `client` 包，server 侧复用同套接口。依赖方向：`client → pkg/endpoint`（接口），实现与接口分离，符合依赖倒置。
- **对象池**：`sync.Pool` 同时用于 `callopt.CallOptions`（client-entry 叶子）与 `timeoutTask`，是 Kitex 降低 GC 压力的惯用手法；注释明确禁止跨 goroutine 持有 `RPCInfo`。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|----------------|
| 客户端中间件链架构图 | `client-middleware-endpoint-architecture.html` | architecture | showcase |
| RPCTimeout 超时时序 | `client-middleware-endpoint-sequence.html` | sequence | showcase |

JSON IR 源文件位于 `json/` 目录。补一张时序图：RPCTimeout 的「主 goroutine Wait / worker goroutine Run / timer 超时」三方交互清晰，适合 sequence 表达。不补 dataflow（无管道）与 lifecycle（无显式状态机，`timeoutTask` 的运行/等待只是 goroutine 调度）。
