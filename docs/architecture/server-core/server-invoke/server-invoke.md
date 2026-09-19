# 请求分发与本地调用（server-invoke）

> 本文是 `server-core` 域下的叶子子系统文档。域级总览见 `../server-core.md`。
> 本文只展开「请求分发到业务 handler、local_caller 本地调用、服务端中间件链最内层」，不展开 NewServer/Run/RegisterService（见 `../server-entry/server-entry.md`）。
>
> 源码基准：`github.com/cloudwego/kitex`，Go 1.20，commit `4fffa48`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| 一元请求分发 `invokeHandleEndpoint` | 从 RPCInfo 取服务名/方法名，查 `services.getService`，`MethodInfo().Handler()` 反射调业务 handler；panic 兜底与 stats 记录 | `server/server.go:347 invokeHandleEndpoint` |
| 流请求分发 `streamHandleEndpoint` | 流版本的分发：包装 `streaming.Args`，gRPC 兼容用 `contextStream` 重写 `Context()` | `server/server.go:390 streamHandleEndpoint` |
| 一元/流分流 `unaryOrStreamEndpoint` | 按 req 是否为 `*streaming.Args` 选择走 `unaryEp` 或 `streamEp` | `server/server.go:324 unaryOrStreamEndpoint` |
| 服务端超时中间件 `serverTimeoutMW` | 读 RPCInfo 的 RPCTimeout，`context.WithTimeout` 叠加到请求 ctx；错误时 cancel | `server/middlewares.go:26 serverTimeoutMW` |
| 进程内调用器 `Invoker`/`NewInvoker` | 不启动网络，用 `invoke.NewIvkTransHandlerFactory` 造内存 transHandler，`Call(msg)` 直接走中间件链 | `server/invoke.go:49 NewInvoker`、`:84 Call` |
| 本地调用器 `LocalCaller`/`NewLocalCaller` | 进程内直接调注册的一元 handler，跳过网络与编解码；`Call(ctx, method, args, result)` 走完整 `s.eps` 中间件链 | `server/local_caller.go:121 NewLocalCaller`、`:152 Call` |
| 方法解析缓存 `cachedResult` | `sync.Map` 缓存 method 串 → svcInfo/MethodInfo/argsType/resultType，重复调用 O(1)；reflect.Type 校验零分配 | `server/local_caller.go:89 cachedResult`、`:222 resolve` |
| 本地 RPCInfo 构造 `newLocalRPCInfo` | 用 `NewRPCInfoWithInlineFields` 构造本地 RI，From 设为 caller + 合成地址 `127.0.0.1:0` | `server/local_caller.go:278` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|-------------|------|------|
| `Invoker` 接口 | `server/invoke.go:37` | `RegisterService` + `Init` + `Call(invoke.Message)` |
| `tInvoker` | `server/invoke.go:43` | 内嵌 `invoke.Handler` 与 `*server`；`Init` 建内存 transHandler，`Call` 转发 |
| `LocalCaller` 接口 | `server/local_caller.go:58` | `Call(ctx, method, args, result)` + `ResolveMethod(ctx, method)` |
| `localCaller` | `server/local_caller.go:110` | 持有 `svr *server`、`traceCtl`、`caller`、`cache sync.Map` |
| `cachedResult` | `server/local_caller.go:89` | 解析结果缓存：methodName/svcInfo/mi/argsType/resultType |
| `gRPCCompatibleServerStream` | `server/server.go:337` 匿名 | 包装 `ServerStream` + `Stream`，兼容旧 streaming 接口 |
| `localCallerAddr` | `server/local_caller.go:36` | 合成地址 `tcp 127.0.0.1:0`，作本地调用的 From 地址 |

## 3. 关键调用链

### 链路一：网络请求分发到业务 handler

1. 传输层（netpoll）解码后调注入的 `s.eps`（`server/server.go:172` 装配的中间件链）。`s.eps` 最外层是 `buildMiddlewares` 装配：流包装 →（可选）`serverTimeoutMW` → 用户 MWBs → `buildCoreMiddleware`（ACL + 错误处理）。
2. 进入 `unaryOrStreamEndpoint`（`server/server.go:324`）：若 req 是 `*streaming.Args` 走 `streamEp`（`streamHandleEndpoint`），否则走 `unaryEp`（`invokeHandleEndpoint`）。
3. `invokeHandleEndpoint`（`server/server.go:347`）：`rpcinfo.GetRPCInfo(ctx)` 取 serviceName/methodName，`s.svcs.getService(serviceName)` 拿 `*service`。defer 中 `recover` panic 转 `ErrPanic` 带堆栈、`rpcStats.SetPanicked`、`rpcinfo.Record(ServerHandleFinish)`、`backup.ClearCtx()`。
4. `MethodInfo().Handler()` 取反射函数，`rpcinfo.Record(ServerHandleStart)`，`backup.BackupCtx(ctx)`，`implHandlerFunc(ctx, svc.getHandler(methodName), args, resp)`（`server/server.go:376`）真正调业务 handler。
5. 错误处理：`kerrors.FromBizStatusError` 命中则 `SetBizStatusErr` 并返回 nil（业务状态错误不抛出）；否则包成 `kerrors.ErrBiz`。

### 链路二：`LocalCaller` 进程内调用

1. `NewLocalCaller(caller, svr)`（`server/local_caller.go:121`）：断言 `svr.(*server)`，加锁 `s.init()` 构建 `s.eps` 中间件链，`check()` 校验有 known 服务，构造 `localCaller`。
2. `Call(ctx, method, args, result)`（`server/local_caller.go:152`）：先 `resolve(method)` 查/缓存 `cachedResult`（`server/local_caller.go:222`，`sync.Map` 命中直接返回）。
3. 校验：`StreamingMode() != StreamingNone` 拒绝流式；`reflect.TypeOf(args/result)` 与缓存类型严格比对（零分配）。
4. `newLocalRPCInfo`（`server/local_caller.go:278`）构造本地 RI，`rpcinfo.NewCtxWithRPCInfo` 挂载，`traceCtl.DoStart` 开 tracing。
5. 直接 `lc.svr.eps(ctx, args, result)`（`server/local_caller.go:200`）——绕过网络与编解码，走完整服务端中间件链。defer `recover` panic、`DoFinish`、`PutRPCInfo` 回收。
6. 成功后从 `ri.Invocation().BizStatusErr()` 取出业务状态错误（中间件链返回 nil 时它被存在 RI 里）。

### 链路三：`Invoker` 内存调用器

1. `NewInvoker(opts...)`（`server/invoke.go:49`）构造 `*server` 并立即 `s.init()` 建链。
2. `Init()`（`server/invoke.go:61`）：`check()` → `initBasicRemoteOption` → 加 meta handler → `newInvokeHandler`（`invoke.go:88`，用 `invoke.NewIvkTransHandlerFactory` 造内存 transHandler，`SetInvokeHandleFunc(s.eps)`，装 inbound/outbound）。
3. `Call(msg invoke.Message)`（`server/invoke.go:84`）转发给 `s.Handler.Call(msg)`，由内存 transHandler 走 `s.eps`。

## 4. 配置项

| option / 配置 | 默认 / 行为 | 位置 |
|---------------|-------------|------|
| `WithEnableContextTimeout` | 开启则 `buildMiddlewares` prepend `serverTimeoutMW` | `server/server.go:155`、`server/middlewares.go:26` |
| RPCInfo 的 RPCTimeout | `serverTimeoutMW` 读取；TTHeader 由 transmeta 设置，gRPC/HTTP2 deadline 已在 ctx 中 | `server/middlewares.go:31-37` |
| `LocalCaller` caller 名 | 作为 RPCInfo 的 From.ServiceName；合成地址 `127.0.0.1:0` | `server/local_caller.go:121/291` |
| LocalCaller 方法串 | 支持 `"ServiceName/MethodName"` 与 `"MethodName"` 两种形式 | `server/local_caller.go:235/262` |
| `TracerCtl` | LocalCaller 用 server 的 TraceController；为 nil 则新建空 TraceController | `server/local_caller.go:140` |

## 5. 错误与重试语义

- **handler panic**：`invokeHandleEndpoint` 与 `localCaller.Call` 均 defer `recover`，转 `kerrors.ErrPanic.WithCauseAndStack` 带堆栈，并 `rpcStats.SetPanicked`。
- **业务错误**：普通 handler 错误包成 `kerrors.ErrBiz`；`BizStatusError` 识别后存在 RI 不抛出（网络路径由传输层序列化回客户端，LocalCaller 路径由 `Call` 末尾从 RI 取出返回）。
- **LocalCaller 校验错误**：method 串解析失败、流式方法不支持、args/result 类型不匹配、依赖 `GenericMethod` 的服务不支持——均返回明确 error，不 panic。
- **serverTimeoutMW**：RPCTimeout > 0 时 `context.WithTimeout`；出错才 `cancel()`（避免正常路径提前 cancel）。
- **Invoker**：`Init` 失败返回 error；`Call` 转发 `Handler.Call` 错误。

## 6. 并发细节

- **goroutine 边界**：LocalCaller/Invoker 复用调用方 goroutine，不额外起 goroutine；网络请求的 goroutine 由 netpoll 拉起。`onServerStart` hooks 在 `Init` 中 `go` 启动。
- **锁**：`NewLocalCaller` 用 `s.Lock()` 保护 `init()`；`localCaller.cache sync.Map` 无锁并发读；`server` 内嵌 `sync.Mutex` 保护 `s.Handler`。
- **对象复用**：`localCaller.cache` 缓存 reflect.Type 实现零分配类型校验；`newLocalRPCInfo` 用 `NewRPCInfoWithInlineFields` 避免池分配；调用结束 `PutRPCInfo(ri)` 回收。
- **context 传播**：LocalCaller 直接复用调用方 ctx（保留其 value/deadline/cancel），只叠 RPCInfo；`serverTimeoutMW` 用 `context.WithTimeout` 叠加服务端超时。
- **stats**：`invokeHandleEndpoint` 用 `rpcinfo.Record` 记录 `ServerHandleStart/Finish`；LocalCaller 用 `traceCtl.DoStart/DoFinish`。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `server/server.go` 的 `invokeHandleEndpoint`/`streamHandleEndpoint`/`unaryOrStreamEndpoint`。
- `server/middlewares.go`：`serverTimeoutMW`。
- `server/invoke.go`：`Invoker`/`tInvoker`/内存 transHandler。
- `server/local_caller.go`：`LocalCaller` 进程内调用。

**Out-of-Scope（不在本仓库源码内）**
- 传输层解码/编码/IO：`pkg/remote/trans/invoke`（内存 transHandler）、netpoll。
- 业务 handler 反射函数本体：IDL 生成代码，不在运行时仓库。
- 中间件链其余成员（用户 MWBs、core middleware 的 ACL/错误处理）：见 server-entry 叶子与 `pkg/acl`。
- netpoll、对端客户端 —— 不在本仓库源码内。

## 8. 与相邻子系统交互

- 上游 → 本叶子：server-entry 叶子的 `buildInvokeChain` 把 `unaryOrStreamEndpoint` 作为最内层 endpoint；传输层经 `SetInvokeHandleFunc(s.eps)` 调它。LocalCaller/Invoker 是本叶子提供的旁路入口，直接调 `s.eps`。
- 本叶子 → 下游：`invokeHandleEndpoint` 经 `MethodInfo().Handler()` 反射调用户业务 handler；`services.getService` 查服务容器（server-entry 叶子）。
- 横向：`serverTimeoutMW` 是本叶子提供的服务端超时中间件，由 `buildMiddlewares` 装配。

## 9. 语言专项适配口径（Go）

- **并发模型**：本叶子是「同步函数反射调用 + 进程内旁路」模型。一元 handler 在调用 goroutine 同步执行；LocalCaller 复用调用方 ctx/goroutine，无额外调度。非 K8s Reconcile。
- **goroutine 生命周期**：本叶子不长期持有 goroutine；`onServerStart` hooks 是唯一后台启动点。LocalCaller 要求 server 未 `Run` 也可用（只需 `init()` 建链）。
- **context 传播**：LocalCaller 保留调用方 ctx 语义（deadline/cancel），只叠 RPCInfo；`serverTimeoutMW` 叠加服务端超时 deadline。这是「业务 ctx 超时 + 服务端配置超时」的双层超时叠加。
- **反射与类型安全**：LocalCaller 用 `reflect.TypeOf` 严格校验 args/result，避免运行时类型不匹配；`sync.Map` 缓存解析结果是典型的「读多写少 + O(1) 热点」优化。
- **internal 边界**：`server` 包复用 `internal/server.Options`；`invoke.Handler` 来自 `pkg/remote/trans/invoke`（内存传输），本叶子只装配。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|----------------|
| 请求分发架构图 | `server-invoke-architecture.html` | architecture | showcase |
| LocalCaller 调用时序 | `server-invoke-sequence.html` | sequence | showcase |

JSON IR 源文件位于 `json/` 目录。补一张时序图：LocalCaller 从 resolve 到 `s.eps` 到业务 handler 的主路径清晰。不补 dataflow（无管道）与 lifecycle（无显式状态机）。
