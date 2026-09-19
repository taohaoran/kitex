# server-core（服务端核心）域总览

> 本域包含以下叶子子系统；各叶子详情见对应文档。
> 源码基准：`github.com/cloudwego/kitex`，Go 1.20，commit `4fffa48`。

## 1. 域职责

server-core 域负责 Kitex 服务端的「入口与配置」与「请求分发与本地调用」两层。`NewServer` 构造 server、注册服务、启动 Run；`invokeHandleEndpoint`/`streamHandleEndpoint` 把网络请求分发到业务 handler，并提供 `LocalCaller`/`Invoker` 两个进程内旁路调用入口，跳过网络与编解码直接走服务端中间件链。

## 2. 叶子索引

| 叶子 | 文档 | 架构图 | 时序图 | 职责一句话 |
|------|------|--------|--------|-----------|
| server-entry | [server-entry.md](server-entry/server-entry.md) | [架构图](server-entry/server-entry-architecture.html) | [时序图](server-entry/server-entry-sequence.html) | NewServer 服务端入口、服务注册、genericserver 泛化服务端、Hooks、Server 配置 |
| server-invoke | [server-invoke.md](server-invoke/server-invoke.md) | [架构图](server-invoke/server-invoke-architecture.html) | [时序图](server-invoke/server-invoke-sequence.html) | 请求分发到业务 handler、local_caller 本地调用、服务端中间件链最内层 |

## 3. 域级机制细节

- **Run 启动流程**：`Run`（`server/server.go:202`）先 init 中间件链，延迟 1s 等待服务注册，再启动传输层监听；`onServerStart` hooks 异步触发。
- **服务容器**：`services.SearchService`/`getService`（`server/service.go`）按服务名查 `*service`，`MethodInfo().Handler()` 反射调业务函数。
- **中间件链**：`buildMiddlewares` 装配流包装 →（可选）`serverTimeoutMW` → 用户 MWBs → core middleware（ACL + 错误处理），最内层是 `unaryOrStreamEndpoint` 分流。
- **进程内旁路**：`LocalCaller`（`server/local_caller.go:121`）要求 server 未 Run 也可用，`sync.Map` 缓存方法解析、reflect.Type 校验 args/result；`Invoker`（`server/invoke.go:49`）用内存 transHandler 跑中间件链。
- **错误处理**：handler panic 转 `ErrPanic` 带堆栈；`BizStatusErr` 存在 RI 不抛出，LocalCaller 末尾从 RI 取出返回。

## 4. 域级图（可选）

本域未单独出域级架构图；两张叶子图已覆盖启动流程与请求分发两个视角，见叶子索引表。
