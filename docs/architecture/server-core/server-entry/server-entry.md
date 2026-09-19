# 服务端入口（server-entry）

> 本文是 `server-core` 域下的叶子子系统文档。域级总览见 `../server-core.md`。
> 本文只展开「NewServer 服务端入口、服务注册、泛化服务端、Hooks、Server 配置」，不展开请求分发到业务 handler 与服务端中间件链（见 `../server-invoke/server-invoke.md`）。
>
> 源码基准：`github.com/cloudwego/kitex`，Go 1.20，commit `4fffa48`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| 服务端构造入口 `NewServer` | 按 `Option...` 构造 `Server`，内部 `internal_server.NewOptions` 装载默认配置 | `server/server.go:76 NewServer` |
| 服务注册 `RegisterService` | 把 `svcInfo` + handler 注册到 `services`；运行中注册直接 panic；重复注册报错 | `server/server.go:176 RegisterService` |
| 服务容器 `services` | 维护 knownSvcMap / fallbackSvc / unknownSvc；支持多服务、fallback、unknown service 动态创建 | `server/service.go:96 services`、`server/service.go:120 addService` |
| 服务查找 `SearchService` | 按 svcName/methodName/strict/codecType 查 svcInfo；gRPC 严格按名查，Thrift 兼容按方法名回退 | `server/service.go:258 SearchService` |
| 启动 `Run` | init → check → 装配 transHandler → `remotesvr.NewServer` → `svr.Start` → 1s 延迟后注册到注册中心 → `waitExit` | `server/server.go:202 Run` |
| 优雅停止 `Stop` | `sync.Once` 保证幂等：执行 shutdown hooks → 注销注册中心 → `svr.Stop` | `server/server.go:271 Stop` |
| 启动/关闭 Hooks | 包级 `onServerStart`/`onShutdown` 钩子切片，`RegisterStartHook`/`RegisterShutdownHook` 注册 | `server/hooks.go:22`、`:29` |
| 服务端中间件链装配 `buildMiddlewares` / `buildInvokeChain` | 装配流包装、serverTimeoutMW、用户 MWBs、core middleware；`endpoint.Chain` 串到 `unaryOrStreamEndpoint` | `server/server.go:142`、`:170` |
| RPCInfo 复用 `initOrResetRPCInfoFunc` | 长连接复用 RPCInfo：pool 开启则 reset 旧 RI，否则内联字段新建 | `server/server.go:108` |
| 限流 bound handler `buildLimiterWithOpt` | 连接数/QPS 限流；非多路复用下在 OnRead 生效以省解码开销 | `server/server.go:494` |
| 泛化服务端 `genericserver.NewServer` | 以 `generic.Service` + `generic.Generic` 构造服务端，自动 `WithGeneric` 并 `RegisterService` | `server/genericserver/server.go:29` |
| 泛化 V2 注册 `genericserver.NewServerV2` / `RegisterService` | 用 `generic.ServiceV2` handler；拒绝旧 `BinaryThriftGeneric`，引导用 V2 | `server/genericserver/server.go:49/59` |
| 注册中心信息构建 `buildRegistryInfo` | 补齐 Addr/ServiceName/PayloadCodec/Weight/Tags，默认权重 `DefaultWeight` | `server/server.go:587` |

对外暴露点：用户业务实现 IDL handler 后调 `server.NewServer(opts...)` + `RegisterService` + `Run()`；`genericserver` 面向不依赖 IDL 的泛化服务端。

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|-------------|------|------|
| `Server` 接口 | `server/server.go:54` | `RegisterService` / `GetServiceInfos` / `Run` / `Stop` |
| `server` 结构体 | `server/server.go:61` | 持有 `*internal_server.Options`、`*services`、`eps endpoint.Endpoint`、`remotesvr.Server`、`isInit/isRun`、`stopped sync.Once`、内嵌 `sync.Mutex` |
| `service` 结构体 | `server/service.go:33` | 单个注册服务：`svcInfo` + `handler` + `unknownMethodHandler` |
| `services` 结构体 | `server/service.go:96` | 服务容器：knownSvcMap、fallbackSvc、nonFallbackSvcs、unknownSvc、binaryThriftGenericV1SvcInfo、combineSvcInfo |
| `unknownService` | `server/service.go:53` | 未知服务动态创建：按 svcName + codecType 懒构造 `BinaryThriftGenericV2`/`BinaryPbGeneric` 的 svcInfo |
| `Hooks` | `server/hooks.go:36` | `[]func()` 切片，`add` 追加 |
| `internal_server.Options` | `internal/server/option.go:131` | 全部服务端运行期配置：Svr/RemoteOpt/Registry/Limit/MWBs/UnaryOptions/StreamOptions/Streaming/EnableContextTimeout/RefuseTraffic |
| `Limit` | `internal/server/option.go:182` | Limits/ConLimit/QPSLimit/LimitReporter/QPSLimitPostDecode |
| `gRPCCompatibleServerStream` | `server/server.go:337` 匿名返回 | 兼容旧 streaming 接口，包装 `ServerStream` + `Stream` |

## 3. 关键调用链

### 链路一：`Run` 启动服务端

1. 用户调 `server.NewServer(ops...)`（`server/server.go:76`），`internal_server.NewOptions` 装默认值（`Registry: registry.NoopRegistry`，`internal/server/option.go:210`）。
2. 用户调 `RegisterService(svcInfo, handler)`（`server/server.go:176`）：加锁，运行中禁止注册；`svcs.addService`（`server/service.go:120`）写入 knownSvcMap 或 fallback/unknown 容器。
3. 用户调 `Run()`（`server/server.go:202`）：置 `isRun=true` → `init()`（`server/server.go:84`，`fillContext` 注入 bus/queue，`buildInvokeChain` 装中间件链）→ `check()`（`server/service.go:155`，至少注册一个服务、方法冲突需 fallback）。
4. `richRemoteOption`（`server/server.go:453`）注入 `SvcSearcher=svcs`、`InitOrResetRPCInfoFunc`、bound handlers（meta + limiter）；`newSvrTransHandler`（`server/server.go:567`）经 `SvrHandlerFactory` 造 transHandler，`SetInvokeHandleFunc(s.eps)` 把业务调用入口注入传输层，再 `NewTransPipeline` 加 inbound/outbound。
5. `remotesvr.NewServer(RemoteOpt, transHdlr)`（`server/server.go:225`）创建底层 RPC server；`svr.Start()` 返回 `errCh`。
6. `waitExit(errCh)`（`server/server.go:622`）：1s 延迟后 `Registry.Register(RegistryInfo)`；select 等退出信号/启动错误。`Stop()` 在退出时 `Deregister` + `svr.Stop`。

### 链路二：服务查找与多服务路由

1. 传输层收到请求后通过 `remoteOpt.SvcSearcher`（即 `s.svcs`）调 `SearchService(svcName, methodName, strict, codecType)`（`server/service.go:258`）。
2. `strict`（gRPC）或 `refuseTrafficWithoutServiceName` 时直接按 `knownSvcMap[svcName]` 查；否则兼容 binary thrift v1/combine service 回退，svcName 为空时按方法名 `searchByMethodName`，否则先按名再 `searchUniqueByMethodName`。
3. 仍未命中且注册了 unknownSvc 时，`unknownSvc.getOrStoreSvc`（`server/service.go:66`）按 codecType 懒建 binary generic 的 svcInfo 并缓存。
4. 命中后 `invokeHandleEndpoint`（`server/server.go:347`）按 `ri.Invocation().ServiceName()` 调 `svcs.getService` 拿 handler，再 `MethodInfo().Handler()` 反射分发（见 server-invoke 叶子）。

### 链路三：泛化服务端构造

1. `genericserver.NewServer(handler, g, opts...)`（`server/genericserver/server.go:29`）：`generic.ServiceInfoWithGeneric(g)` 造 svcInfo，追加 `server.WithGeneric(g)`，转调 `server.NewServer`。
2. 立即 `svr.RegisterService(svcInfo, handler)`（`server/genericserver/server.go:41`），失败 panic。
3. `NewServerV2`（`:49`）用 `*generic.ServiceV2` handler，经 `RegisterService`（`:59`）注册；若检测到旧 `BinaryThriftGeneric` extra 直接 panic 引导迁移到 V2。

## 4. 配置项

| option / 配置 | 默认 / 行为 | 位置 |
|---------------|-------------|------|
| `WithServiceName` | 服务端服务名，注册中心使用 | `server/option.go` |
| `WithListener` / `WithAddr` | 监听地址；`Run` 中 `Proxy.Replace` 可替换 | `server/server.go:211` |
| `WithRegistry` | 服务注册中心；默认 `NoopRegistry` | `internal/server/option.go:210` |
| `WithMiddleware` / `WithMiddlewareBuilder` | 服务端通用中间件，排在 core middleware 之前 | `server/option.go`、`server/server.go:159` |
| `WithUnaryMiddleware` / `WithStreamMiddleware` | 一元/流专用中间件 | `server/option_unary.go`、`server/option_stream.go` |
| `WithLimit` | 连接数/QPS 限流；`buildLimiterWithOpt` 组装 bound handler | `server/server.go:494` |
| `WithGeneric(g)` | 注入泛化序列化器 | `server/genericserver/server.go:37` |
| `WithRefuseTrafficWithoutServiceName` | 拒绝无服务名流量 | `internal/server/option.go:172` |
| `WithEnableContextTimeout` | 开启则 prepend `serverTimeoutMW`（见 server-invoke 叶子） | `server/server.go:155` |
| `RegisterOption`：IsFallbackService / IsUnknownService | 标记 fallback/unknown 服务 | `server/service.go:131/122`、`server/register_option.go` |
| `WithExitSignal` | 退出信号源，`waitExit` 监听 | `server/server.go:623` |

## 5. 错误与重试语义

- **注册期错误**：`RegisterService` 对 nil svcInfo/handler、运行中注册、重复注册、多 fallback、方法名冲突且无 fallback 均直接 panic（`server/server.go:180-194`、`server/service.go:132/138/211`）——属于编程错误。
- **启动期错误**：`check()` 无服务或方法冲突返回 error；`remotesvr.NewServer`/`svr.Start` 失败 `Run` 返回错误并记日志。
- **注册中心错误**：`waitExit` 中 `Registry.Register` 失败会导致 `Run` 返回（`server/server.go:635`）；Stop 时 `Deregister` 错误合并到返回值。
- **handler panic**：`invokeHandleEndpoint` 的 defer `recover`（`server/server.go:358`）把业务 panic 转 `kerrors.ErrPanic` 并带堆栈，同时 `rpcStats.SetPanicked`。
- **业务错误**：`kerrors.FromBizStatusError` 识别业务状态错误，成功时 `SetBizStatusErr` 返回 nil；普通业务错误包成 `kerrors.ErrBiz`。
- **Stop 幂等**：`stopped sync.Once` 保证多次 Stop 只执行一次。

## 6. 并发细节

- **goroutine 启停**：`Run` 中 `onServerStart` hooks 用 `go onServerStart[i]()` 异步启动（`server/server.go:253`）；profiler 用 `gofunc.GoFunc` 启动（`server/server.go:235`）；真正的连接 accept/请求处理 goroutine 由 `remotesvr.Server`（netpoll）管理。本叶子不直接持有长生命周期 goroutine。
- **锁**：`server` 内嵌 `sync.Mutex`，`RegisterService`/`Run`/`Stop`/`buildRegistryInfo` 用 `s.Lock` 保护 `isRun`/`svr`/`RegistryInfo`；`unknownService.mutex sync.RWMutex` 保护动态服务 map；`muStartHooks`/`muShutdownHooks` 保护 hooks 切片。
- **RPCInfo 复用**：`initOrResetRPCInfoFunc`（`server/server.go:108`）长连接下 reset 而非新建 RI，降低分配；`rpcinfo.PoolEnabled()` 控制走 reset 还是内联新建路径。
- **context 传播**：`fillContext`（`server/server.go:101`）把 bus/queue 注入构造期 ctx 供 `MiddlewareBuilder`；请求级 ctx 由传输层创建并经 `rpcinfo.NewCtxWithRPCInfo` 挂载 RI。
- **backup session**：`invokeHandleEndpoint` 中 `backup.BackupCtx(ctx)` / `backup.ClearCtx()`（`server/server.go:370/375`）在 handler 前后做 localsession 上下文备份/清理。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `server/server.go`：Server 接口、server 实现、NewServer/Run/Stop/RegisterService/buildInvokeChain/buildMiddlewares/limiter。
- `server/service.go`：services 容器、服务查找、unknown service 动态创建。
- `server/hooks.go`：启动/关闭 hooks。
- `server/genericserver/`：泛化服务端工厂。
- `internal/server/`：Options 结构体与默认值。

**Out-of-Scope（不在本仓库源码内）**
- 请求分发到 handler 的具体调用链、local_caller、服务端中间件链细节：见 `server-invoke` 叶子。
- 传输层 accept/解码/连接管理：`pkg/remote/remotesvr`、netpoll。
- 限流算法本身：`pkg/limiter`（本叶子只组装 bound handler）。
- 注册中心实现：`pkg/registry` 与外部注册中心。
- 业务 handler（IDL 生成或用户实现）：不在运行时仓库。

## 8. 与相邻子系统交互

- 上游 → 本叶子：用户业务调 `server.NewServer` + `RegisterService` + `Run`；传输层经 `remoteOpt.SvcSearcher`（= `services`）回调 `SearchService` 解析请求属于哪个服务。
- 本叶子 → 下游：`newSvrTransHandler` 把 `s.eps`（中间件链）通过 `SetInvokeHandleFunc` 注入传输层；`remotesvr.NewServer` 启动 netpoll 监听。
- 横向：`server-invoke` 叶子实现 `invokeHandleEndpoint`/`streamHandleEndpoint`/`local_caller`，是 `s.eps` 链的最内层。
- 注册中心侧：`waitExit` 延迟 1s 后 `Registry.Register`，Stop 时 `Deregister`。

## 9. 语言专项适配口径（Go）

- **并发模型**：服务端是「事件驱动 + 每请求 goroutine」模型，非 K8s Reconcile。accept 与 IO 复用由 netpoll（外部依赖，不在本仓库运行时核心）承担；业务 handler 在 netpoll 拉起的 goroutine 中经中间件链执行。本叶子只负责把 `eps` 闭包交给传输层。
- **goroutine 边界**：`onServerStart` hooks 与 profiler 用 `gofunc.GoFunc` 起后台 goroutine；请求处理 goroutine 生命周期由传输层管理，本叶子的 `invokeHandleEndpoint` defer 负责 panic 兜底与 stats 记录。
- **context 传播**：构造期 ctx（bus/queue）与请求期 ctx（RPCInfo）分离；`initOrResetRPCInfoFunc` 按长连接 reset RI 是性能关键。
- **internal 边界**：`internal/server` 隔离实现，`server` 包 type alias 再导出；依赖方向 `server → internal/server → pkg/remote/remotesvr`。`pkg/endpoint` 接口被 server/client 两侧复用。
- **多二进制**：运行时库无 `cmd/` 入口；唯一二进制 `tool/cmd/kitex`（代码生成）与本叶子无关。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|----------------|
| 服务端启动架构图 | `server-entry-architecture.html` | architecture | standard（showcase 布局校验多次不通过，已删重叠标签并回退 standard） |
| Run 启时序 | `server-entry-sequence.html` | sequence | showcase |

JSON IR 源文件位于 `json/` 目录。补一张时序图：`Run` 从 init 到注册中心注册的主流程清晰。不补 dataflow（无 ETL）与 lifecycle（无显式状态机，`isInit/isRun/stopped` 仅布尔/Once）。
