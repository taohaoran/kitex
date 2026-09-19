# 客户端入口（client-entry）

> 本文是 `client-core` 域下的叶子子系统文档。域级总览见 `../client-core.md`。
> 本文只展开「NewClient 客户端入口、配置装载、泛化客户端、单次调用级选项」，不重复展开客户端中间件链编织与 rpctimeout（见 `../client-middleware-endpoint/client-middleware-endpoint.md`），也不展开流式 stub（见 `../../streaming/stream-client-server/stream-client-server.md`）。
>
> 源码基准：`github.com/cloudwego/kitex`，Go 1.20，commit `4fffa48`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| 标准客户端构造入口 `NewClient` | 由代码生成产物（IDL stubs）调用，按 `serviceinfo.ServiceInfo` + `Option...` 构造可发起 RPC 的 `Client` | `client/client.go:96 NewClient` |
| 客户端初始化流水线 `init` | 传输协议初始化、选项校验、熔断器/重试/代理/连接池/LB 缓存初始化、中间件链编织、预热 | `client/client.go:114 (*kClient).init` |
| 核心调用方法 `Call` | 合并备份上下文、校验状态、初始化 RPCInfo、开启 tracing、按重试容器执行、fallback | `client/client.go:365 (*kClient).Call` |
| 内部终态化包装 `kcFinalizerClient` | 解决 `kClient` 构造 endpoint 时的循环引用，把 `runtime.SetFinalizer` 挂在包装层，GC 时自动 `Close` | `client/client.go:86 kcFinalizerClient` |
| 泛化客户端 `genericclient.NewClient` | 不依赖 IDL 生成代码，直接以 `generic.Generic` 序列化器构造客户端，支持 `GenericCall` 与四种流式泛化 API | `client/genericclient/client.go:38 NewClient` |
| 泛化服务客户端实现 `genericServiceClient` | 把 `GenericCall` 适配为 `kClient.Call`，请求/响应统一包装为 `generic.Args` / `generic.Result` | `client/genericclient/client.go:82 genericServiceClient` |
| 单次调用级选项 `callopt.Option` | 每次调用可覆盖目标地址、超时、重试、fallback、压缩器等；用 `sync.Pool` 复用 `CallOptions` | `client/callopt/options.go:76 Option` |
| 调用选项装载 `callopt.Apply` | 从 ctx 取出本次调用选项，应用到 RPCConfig/RemoteInfo，并记录 debug 字符串 | `client/callopt/options.go:271 Apply` |
| 客户端级选项 `client.Option` | 传输协议、目标服务、服务发现/负载均衡、连接池、重试、熔断、fallback、tracing、xDS、预热等全部构造期配置 | `client/option.go:74` 起各 `With*` 函数 |
| 选项结构体 `internal/client.Options` | 真正持有全部运行期配置的结构体；公开 `client.Option` 仅是其函数式别名（type alias） | `internal/client/option.go:148 Options` |
| 选项应用与远端 option 装配 | `NewOptions` 应用全部 `Option`，并按传输协议选择连接池/handler 工厂（netpoll / nphttp2 / ttstream） | `internal/client/option.go:230 NewOptions`、`internal/client/option.go:278 initRemoteOpt` |
| 预热 `warmingUp` | 可选的服务发现与连接池预热，按 `warmup.ErrorHandling` 策略处理失败 | `client/client.go:601 (*kClient).warmingUp` |
| 关闭 `Close` | 依次执行 `CloseCallbacks`（连接池、retry container、lbf hook 注销）、关闭熔断套件；panic 被 recover | `client/client.go:550 (*kClient).Close` |

对外暴露点：用户代码（IDL 生成的 stubs）与 `genericclient` 包只依赖 `client.Client` 接口与 `client.NewClient`；`callopt.Option` 随每次 RPC 传入。

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|-------------|------|------|
| `Client` 接口 | `client/client.go:67` | 对外核心抽象，`Call(ctx, method, request, response) error`，设计给生成代码使用 |
| `kClient` 结构体 | `client/client.go:71` | 客户端内部实现：持有 `svcInfo`、一元 endpoint `eps`、流 endpoint `sEps`、`*client.Options`、`lbf`、`inited/closed` 标志 |
| `kcFinalizerClient` | `client/client.go:86` | 包装 `*kClient`，解决循环引用以便挂 finalizer；`Call` 里 `defer runtime.KeepAlive(kf)` |
| `middleware` 结构体 | `client/client.go:278` | 汇总四类中间件切片：通用 `mws`、一元 `uMws`、流 `smws`/`sMws`，在 `initMiddlewares` 中按固定顺序装配 |
| `genericclient.Client` 接口 | `client/genericclient/client.go:69` | 泛化客户端接口：`GenericCall` + 客户端流/服务端流/双向流三个工厂方法 + `Close` |
| `genericServiceClient` | `client/genericclient/client.go:82` | 持有 `kClient`、`sClient client.Streaming`、`generic.Generic`，并缓存 `isBinaryGeneric` / `getMethodFunc` 两个 extra |
| `callopt.CallOptions` | `client/callopt/options.go:40` | 单次调用的可变配置快照：`configs`、`svr`、`locks`、`RetryPolicy`、`Fallback`、`CompressorName` 等；`sync.Pool` 复用 |
| `callopt.Option` | `client/callopt/options.go:76` | 单次调用选项的函数式封装 `func(o *CallOptions, di *strings.Builder)`；`F()`/`NewOption()` 用于与 streamcall 互转 |
| `client.Option`（= `client.Option`） | `client/option.go:50`、`internal/client/option.go:225` | 构造期选项 type alias：`F func(o *client.Options, di *utils.Slice)` |
| `client.Options` | `internal/client/option.go:148` | 全部运行期配置（Cli/Svr/Configs/Locks/RemoteOpt/Proxy/Resolver/Balancer/CBSuite/MWBs/Bus/Events/TracerCtl…） |
| `client.UnaryOptions` / `StreamOptions` | `internal/client/option.go:68`、`:103` | 仅对一元方法 / 仅对流方法生效的子配置与中间件切片 |
| `Suite` 接口 | `client/option.go:69` | 把一组相关 `Option` 打包成一个，保持顺序与存在性 |

## 3. 关键调用链

### 链路一：`NewClient` 构造到可调用状态

1. 用户（或 IDL 生成代码）调用 `client.NewClient(svcInfo, opts...)`（`client/client.go:96`）。先校验 `svcInfo != nil`，构造 `kcFinalizerClient{ kClient: &kClient{} }`。
2. `kc.opt = client.NewOptions(opts)`（`client/client.go:102`；实现在 `internal/client/option.go:230`）：生成默认 `Options`，默认传输协议为 `transport.Framed`，默认长连接池 `MaxIdlePerAddress:10 / MaxIdleGlobal:100 / MaxIdleTimeout:1min`，再依次 `Apply` 全部用户 `Option`，最后 `initTraceController` + `initRemoteOpt`（按是否 GRPC/TTHeaderStreaming 选择连接池与 handler 工厂）。
3. 进入 `kc.init()`（`client/client.go:114`）：`initTransportProtocol` → `checkOptions`（必须有 `Svr.ServiceName`）→ `initCircuitBreaker` → `initRetryer` → `initProxy` → `initConnPool` → `initLBCache` → `initContext` → `initMiddlewares` → `initDebugService` → `richRemoteOption` → `buildInvokeChain` → `warmingUp`。
4. `buildInvokeChain`（`client/client.go:479`）先取 `invokeHandleEndpoint`（`client/client.go:506`，内部 `newCliTransHandler` 装配 inbound/outbound handler 流水线），再用 `endpoint.Chain(mw.mws...)(innerHandlerEp)` 与 `endpoint.UnaryChain(mw.uMws...)` 把中间件包成 `kc.eps`；流式同理得到 `kc.sEps`。
5. 最后 `runtime.SetFinalizer(kc, ...Close)`（`client/client.go:108`），返回 `Client`。`init()` 失败会 `Close` 回滚。

### 链路二：一次一元 RPC 的 `Call`

1. 生成的 stub 调用 `kc.Call(ctx, method, request, response)`（`client/client.go:365`）。先 `backup.RecoverCtxOnDemands` 合并服务端备份的上下文，再 `validateForCall`（未初始化/已关闭/nil ctx 直接 panic）。
2. `initRPCInfo`（`client/client.go:767`）：clone RPCConfig、构造 `remoteinfo.NewRemoteInfo(svr, method)`、`applyCallOptions`（`client/client.go:353`，从 ctx 取 `callopt.Option` 并 `callopt.Apply`）、构造 `rpcinfo.NewRPCInfo`，按 method 设置 oneway/streaming、按 `Timeouts` provider 覆盖超时，最后 `rpcinfo.NewCtxWithRPCInfo`。
3. `TracerCtl.DoStart` 开启 tracing；`retry.PrepareRetryContext(ctx)` 做重试上下文隔离。
4. 校验 `MethodInfo` 存在且为一元方法后：无重试容器则直接 `kc.eps(ctx, req, resp)`（`client/client.go:410`）；有重试容器则 `RetryContainer.WithRetryIfNeeded(...)` 包装 `rpcCallWithRetry`（`client/client.go:429`），每次重试 `atomic.AddInt32` 计数并重建 RPCInfo。
5. `doFallbackIfNeeded`（`client/client.go:708`）按 callopt > client 优先级取 fallback policy；defer 中 `recover` panic 转 `ClientPanicToErr`、`TracerCtl.DoFinish`、按需 `PutRPCInfo` 回收、`callOpts.Recycle()` 归还池。

### 链路三：泛化客户端 `GenericCall`

1. `genericclient.NewClient(destService, g, opts...)`（`client/genericclient/client.go:38`）：`generic.ServiceInfoWithGeneric(g)` 造 svcInfo，强制追加 `WithGeneric`、`WithDestService`、`WithTransportProtocol(TTHeaderStreaming)`，转调 `client.NewClient`。
2. 包装为 `genericServiceClient`，缓存 `g.GetExtra(IsBinaryGeneric)` 与 `GetMethodNameByRequestFunc`；挂 `SetFinalizer(Close)`。
3. `GenericCall(ctx, method, request, ...callopt)`（`client/genericclient/client.go:93`）：`NewCtxWithCallOptions` 注入单次选项；binary 泛化时注入 `StreamingNone`；http 泛化用 `getMethodFunc(request)` 反查 method；用 `svcInfo.MethodInfo(method)` 造 `generic.Args{Method, Request}` 与 `generic.Result`，转调 `gc.kClient.Call`，成功后 `_result.GetSuccess()` 返回业务响应。
4. 流式泛化（`ClientStreaming`/`ServerStreaming`/`BidirectionalStreaming`，`client/genericclient/client.go:131/145/170`）：`NewCtxWithCallOptions` + binary 模式注入对应 StreamingMode，`gc.sClient.StreamX(ctx, method)` 拿到底层 `streaming.ClientStream`，再封装为三种泛化流客户端。

## 4. 配置项

| option / 配置 | 默认 / 行为 | 位置 |
|---------------|-------------|------|
| `WithDestService(svr)` | 目标服务名，`OnceOrPanic` 只允许设一次；`checkOptions` 要求非空 | `client/option.go:132`、`client/client.go:148` |
| `WithTransportProtocol(tp)` | 默认 `transport.Framed`；`initTransportProtocol` 对 Protobuf 非 gRPC 自动降为 `TTHeaderFramed` | `client/option.go:74`、`client/client.go:592`、`internal/client/option.go:250` |
| `WithHostPorts(...)` | 注入 `SynthesizedResolver` 固定实例，覆盖服务发现 | `client/option.go:142` |
| `WithResolver` / `WithLoadBalancer` / `WithLoadBalancer` | 自定义 resolver / 负载均衡器；`initLBCache` 无 resolver 时造假 resolver 以支持 `callopt.WithHostPort` 事后指定 | `client/option.go:181`、`client/client.go:236` |
| `WithLongConnection(cfg)` / `WithShortConnection` | 默认长连接池 `MaxIdlePerAddress:10/MaxIdleGlobal:100/MaxIdleTimeout:1min`；零 `IdleConfig` 表示短连接 | `client/option.go:208/199`、`internal/client/option.go:310` |
| `WithRPCTimeout` / `WithUnaryRPCTimeout` / `WithConnectTimeout` | 设置 RPC/连接超时并置对应 Locks 位；`WithTimeoutProvider` 优先级最高（先应用） | `client/option.go:249/259`、`client/option_unary.go:43` |
| `WithFailureRetry` / `WithBackupRequest` / `WithMixedRetry` / `WithRetryMethodPolicies` | 一元重试策略；三者互斥，`RetryMethodPolicies` 优先级高于通配 | `client/option.go:342/363/384/406` |
| `WithFallback` | 一元失败 fallback 策略；`IsPolicyValid` 校验 | `client/option.go:453` |
| `WithCircuitBreaker(suite)` | 注入熔断套件；`initCircuitBreaker` 把事件总线/队列塞给 suite | `client/option.go:464`、`client/client.go:155` |
| `WithWarmingUp(wuo)` | 构造末尾做服务发现/连接池预热；`ErrorHandling` 支持 IgnoreError/WarningLog/ErrorLog/FailFast | `client/option.go:574`、`client/client.go:601` |
| `WithXDSSuite(suite)` | 开启 xDS，注入 router middleware 与 resolver | `client/option.go:582` |
| `WithSuite` / `TailOption` / `WithContextBackup` | 选项套件 / 尾部选项 / localsession 上下文备份 | `client/option.go:86/607/596` |
| `callopt.WithHostPort` / `WithURL` / `WithRPCTimeout` / `WithRetryPolicy` / `WithFallback` / `WithGRPCCompressor` | 单次调用覆盖目标、超时、重试、fallback、压缩器；`Apply` 时写入 `CallOptions` | `client/callopt/options.go:95/120/148/201/224/237` |
| `callopt.WithBinaryGenericIDLService` | binary 泛化指定目标 IDL 服务名，仅对 BinaryThriftGenericV2/BinaryPbGeneric 生效 | `client/callopt/options.go:258` |

## 5. 错误与重试语义

- **构造期错误**：`NewClient` 任一步 `init*` 失败都会 `Close()` 回滚并返回 error（`client/client.go:103-106`）；`checkOptions` 缺服务名直接报错。
- **调用期状态校验**：`validateForCall`（`client/client.go:687`）对未初始化/已关闭/nil ctx 直接 `panic`，属于编程错误防护而非业务错误。
- **业务 panic 保护**：`Call` 的 defer 中 `recover` 把 panic 转为 `rpcinfo.ClientPanicToErr`（`client/client.go:378`），并在 `Close` 中 recover 关闭期 panic（`client/client.go:552`）。
- **重试**：`RetryContainer.WithRetryIfNeeded` 负责失败重试/备份请求；`rpcCallWithRetry`（`client/client.go:429`）用 `atomic.AddInt32` 记录次数、`atomic.Value` 保存上一次 RPCInfo 供 `r.Prepare`；`callopt.WithRetryPolicy` 可在调用期临时启用（`client/client.go:396` 懒初始化 retry container）。
- **fallback**：`doFallbackIfNeeded`（`client/client.go:708`）callopt 优先于 client option；`reportAsFallback` 决定上报口径。
- **连接层错误包装**：由 `client-middleware-endpoint` 叶子的 `newIOErrorHandleMW`/`DefaultClientErrorHandler` 统一包装为 `kerrors.ErrRemoteOrNetwork`，本叶子不展开。
- **RPCInfo 回收**：`recycleRI` 标志控制是否 `PutRPCInfo`——无重试且成功才回收；重试成功/备份请求场景不能回收（`client/client.go:383-391` 注释）。

## 6. 并发细节

- **goroutine 启停**：`Call` 本身在用户 goroutine 内同步执行；真正的 IO goroutine 由 `remotecli.NewClient`/连接池与传输层（netpoll/gonet/nphttp2）管理，本叶子只持有 endpoint 闭包。`warmingUp` 在 `init` 阶段同步执行。
- **重试并发**：备份请求（backup request）模式下 retry container 会在新 goroutine 发起兜底请求；`rpcCallWithRetry` 用 `atomic.Int32`/`atomic.Value` 跨 goroutine 安全记录调用次数与 prevRI。
- **RPCInfo/CallOptions 复用**：`rpcinfo.PutRPCInfo` 与 `callOptionsPool sync.Pool`（`client/callopt/options.go:36`）对象池复用，`Recycle()` 置零后归还；注释明确「持有 RPCInfo 在新 goroutine 中是禁止的」。
- **finalizer**：`kcFinalizerClient` 用 `runtime.SetFinalizer` 在 GC 兜底 `Close`，`Call` 中 `defer runtime.KeepAlive(kf)` 防止提前 finalizer；这是为绕开构造 endpoint 时的循环引用（`client/client.go:83-93`）。
- **context 传播**：`initContext`（`client/client.go:172`）把 event bus/queue/`ExtraTimeout` 注入构造期 ctx，供 `MiddlewareBuilder` 读取；每次 `Call` 新建 `rpcinfo.NewCtxWithRPCInfo` 把单次 RPCInfo 挂到调用 ctx。超时取消由 `rpctimeoutMW`（见相邻叶子）驱动 `workerPool`。
- **Close 并发**：`Close` 明确标注「not concurrency safe」，靠 `closed` bool 幂等。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `client/client.go`：Client 接口、kClient 实现、NewClient/init/Call/Close/buildInvokeChain。
- `client/genericclient/`：泛化客户端接口与三类流式泛化客户端封装。
- `client/callopt/`：单次调用选项定义、Apply、CallOptions 池化。
- `client/option*.go` + `internal/client/option*.go`：构造期选项函数与 `Options` 结构体。

**Out-of-Scope（不在本仓库源码内）**
- 传输与编解码：`pkg/remote`、netpoll、nphttp2/grpc、thrift/protobuf codec —— 由 `invokeHandleEndpoint` 经 `remotecli` 调用，不在本叶子展开。
- 服务发现/负载均衡实现：`pkg/discovery`、`pkg/loadbalance`、`lbcache` —— 本叶子只持有 `Resolver/Balancer/lbf` 句柄，具体实例挑选在 `client-middleware-endpoint` 叶子的 resolve 中间件中。
- 重试/熔断/fallback 具体策略引擎：`pkg/retry`、`pkg/circuitbreak`、`pkg/fallback` —— 本叶子只做接线与调用点。
- 中间件链编织细节（resolve/CB/acl/context/rpctimeout）与流式 stub 本体：分别见 `client-middleware-endpoint` 与 `stream-client-server` 叶子。
- 外部网络库 netpoll、对端 RPC 服务 —— 不在本仓库源码内。

## 8. 与相邻子系统交互

- 上游 → 本叶子：IDL 生成的 stub（`kitex` 工具生成，不在本运行时仓库）或 `genericclient` 用户调用 `Client.Call` / `GenericCall` / 流工厂；`callopt.Option` 经 `NewCtxWithCallOptions` 从 ctx 注入。
- 本叶子 → 下游中间件：`buildInvokeChain` 把 `kClient.eps`/`sEps` 交给 `client-middleware-endpoint` 叶子编织的中间件链（context MW → 用户 MW → acl → resolve → instance CB → IMWB → IO error handle；一元再加 service CB/xDS/rpctimeout）。
- 本叶子 → 传输层：`invokeHandleEndpoint` 经 `remotecli.NewClient` + `newCliTransHandler`（`client/client.go:506/577`）装配 inbound/outbound handler，最终落到 `pkg/remote/trans` 的 netpoll/gonet/nphttp2 实现。
- 横向：`internal/client` 的 `Options` 被 `client` 包以 type alias 再导出，保证公开 API 与内部实现同源。

## 9. 语言专项适配口径（Go）

- **并发模型**：本叶子是「同步调用 + 对象池 + finalizer」模型，不是 K8s Reconcile/informer 模型。`Call` 在用户 goroutine 同步阻塞等待结果；IO 异步性下沉到传输层（netpoll eventloop）与 retry container 的 backup goroutine。context 超时取消经 `rpcTimeoutMW`→`workerPool` 传播到下层 endpoint（见相邻叶子）。
- **goroutine 边界**：本叶子自身不长期持有 goroutine；唯一后台 ticker 在 `rpctimeout_pool.go`（相邻叶子）。`warmingUp` 同步。资源生命周期由 `CloseCallbacks` + `SetFinalizer` 双保险管理。
- **internal 边界与依赖方向**：`internal/client` 是隔离实现，`client` 包通过 type alias（`type Option = client.Option`）再导出，外部无法直接 import `internal/client`；依赖方向单向 `client → internal/client → pkg/*`，无环。`pkg/endpoint` 只定义接口（`Endpoint/Middleware/Chain`），客户端中间件实现在 `client` 包，符合依赖倒置（接口在消费方一侧的 `pkg/endpoint`）。
- **多二进制**：本项目运行时库无 `cmd/` 二进制入口；唯一二进制是代码生成工具 `tool/cmd/kitex`，与本叶子无关。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|----------------|
| client-entry 架构图 | `client-entry-architecture.html` | architecture | standard（showcase 多次布局校验不通过：竖向连线标签压源节点，已删除非关键连线标签并回退 standard） |
| 一元 RPC 调用时序 | `client-entry-sequence.html` | sequence | showcase |

JSON IR 源文件位于 `json/` 目录（`client-entry-architecture.json`、`client-entry-sequence.json`）。本叶子补一张时序图：`Call` 从入口到 transport handler 的主调用链清晰，适合时序表达；不补 dataflow（无 ETL/管道）与 lifecycle（无显式状态机，`inited/closed` 只是布尔标志）。
