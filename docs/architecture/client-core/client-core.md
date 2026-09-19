# client-core（客户端核心）域总览

> 本域包含以下叶子子系统；各叶子详情见对应文档。
> 源码基准：`github.com/cloudwego/kitex`，Go 1.20，commit `4fffa48`。

## 1. 域职责

client-core 域负责 Kitex 客户端的「入口与配置」与「中间件链与端点来源」两层。`NewClient` 是用户构造客户端的唯一入口，负责把 callopt/option/generic 配置装载进 `kClient`，并把用户中间件、rpcinfo 透传、RPCTimeout 等编织成一条从调用方到传输层的 endpoint 链。负载均衡作为 endpoint 的实例来源，其算法实现归另一分片，本域只讲中间件链与 resolve 机制。

## 2. 叶子索引

| 叶子 | 文档 | 架构图 | 时序图 | 职责一句话 |
|------|------|--------|--------|-----------|
| client-entry | [client-entry.md](client-entry/client-entry.md) | [架构图](client-entry/client-entry-architecture.html) | [时序图](client-entry/client-entry-sequence.html) | NewClient 客户端入口、配置装载、genericclient 泛化客户端、callopt 单次调用级选项 |
| client-middleware-endpoint | [client-middleware-endpoint.md](client-middleware-endpoint/client-middleware-endpoint.md) | [架构图](client-middleware-endpoint/client-middleware-endpoint-architecture.html) | [时序图](client-middleware-endpoint/client-middleware-endpoint-sequence.html) | 客户端中间件链编织、rpcinfo/context 透传、RPCTimeout 中间件、endpoint 接口 |

## 3. 域级机制细节

- **中间件链固定顺序**：`initMiddlewares`（`client/client.go:297`）按固定顺序 prepend——context 中间件 → 用户 MWB → 实例中间件 → RPCTimeout → 错误处理；`buildInvokeChain`（`client/client.go:479`）把链封成 `Call` 最终调用传输层。
- **单次调用选项 callopt**：`callopt.Options` 在 `Call` 时通过 `context.WithValue` 叠到请求 ctx，与客户端级 option 合并；`GenericCall` 走 genericclient 旁路。
- **RPCTimeout 中间件**：`rpctimeout.go` 用 worker pool（默认 128 worker、1 分钟任务缓冲）跑业务调用，超时即取消；与 client-entry 的超时配置项衔接。
- **endpoint 来源**：`pkg/endpoint` 定义 `Endpoint`/`Middleware`/`MiddlewaresBuilder` 接口；resolve 选实例 maxRetry=6，负载均衡算法归另一分片。

## 4. 域级图（可选）

本域未单独出域级架构图；两张叶子图已覆盖入口与中间件链两个视角，见叶子索引表。
