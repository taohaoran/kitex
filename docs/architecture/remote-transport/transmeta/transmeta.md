# 传输元信息透传与自定义元信息 handler（transmeta）

> 本文是 `remote-transport` 域下的叶子子系统文档。域级总览见 `../remote-transport.md`。
> 本文只展开"TTHeader/HTTP2 元信息透传、metainfo 跨进程序列化、bound 元信息 handler 装配、自定义元信息 handler 工厂"，不展开协议头字节编解码（见 `../codec-payload/codec-payload.md`）与流传输实现（见 `transport-implementations/ttstream-mux`、`nphttp2-grpc`）。
>
> 源码基准：`github.com/cloudwego/kitex`，go 1.20，commit `4fffa48`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| 透传元信息键常量 | mesh header 整型键（LogID/FromService/ToService/SpanContext 等）与字符串键（isn/rip/tc/ti、性能打点、crc32c） | `pkg/remote/transmeta/metakey.go:24/53` |
| gRPC HTTP2 头键 | destination-service/source-service/stream-log-id 等（小写，遵循 http2 规范） | `pkg/remote/transmeta/http_metakey.go:24` |
| metainfo 客户端 handler | 单例 `MetainfoClientHandler`：WriteMeta 把 ctx metainfo 写入 TransInfo，ReadMeta 回读 backward 值 | `pkg/transmeta/metainfo.go:33/68/77` |
| metainfo 服务端 handler | 单例 `MetainfoServerHandler`：ReadMeta 把 TransInfo 还原进 ctx，WriteMeta 回写 backward 值 | `pkg/transmeta/metainfo.go:34/88/126` |
| TTHeader 元信息 handler | `ClientTTHeaderHandler`/`ServerTTHeaderHandler`：在 TTHeader 协议上读写业务键与 BizStatusErr | `pkg/transmeta/ttheader.go:46/47` |
| HTTP2 元信息 handler | `ClientHTTP2Handler`/`ServerHTTP2Handler`：在 gRPC/http2 头上透传，兼为 StreamingMetaHandler | `pkg/transmeta/http2.go:31/75` |
| 元信息传播模式 | `MetainfoPropagationMode` 全局/按 ctx 策略控制前向透传 | `pkg/transmeta/metainfo_propagation.go:30/103` |
| bound 元信息 handler 装配 | `NewTransMetaHandler([]MetaHandler)` 把多个 MetaHandler 串成 `DuplexBoundHandler` 挂入管道 | `pkg/remote/bound/transmeta_bound.go:30` |
| 自定义元信息 handler 工厂 | `NewCustomMetaHandler` 用函数式选项只实现需要的部分方法 | `pkg/remote/custom_meta_handler.go:39` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|------|------|------|
| `remote.MetaHandler` | `pkg/remote/trans_meta.go:24` | WriteMeta/ReadMeta 两阶段元信息读写接口（由本域各 handler 实现） |
| `remote.StreamingMetaHandler` | `trans_meta.go:30` | OnConnectStream/OnReadStream，流建立/首帧前的元信息读写 |
| `transMetaHandler`（bound） | `bound/transmeta_bound.go:34` | 持有 `[]MetaHandler`，在 Write/OnMessage 中依次调用各 handler 的 WriteMeta/ReadMeta，串入 TransPipeline |
| `metainfoClientHandler`/`metainfoServerHandler` | `metainfo.go:42/86` | metainfo 透传单例；区分 gRPC（经 http2 metadata）与 TTHeader（经 TransInfo）路径 |
| `clientTTHeaderHandler`/`serverTTHeaderHandler` | `ttheader.go:51/150` | TTHeader 协议元信息 handler；`ParseBizStatusErr` 解析业务状态错误 |
| `clientHTTP2Handler`/`serverHTTP2Handler` | `http2.go:33/77` | HTTP2/gRPC 元信息 handler |
| `customMetaHandler` | `custom_meta_handler.go:76` | 函数式选项构造的可空方法 MetaHandler，未实现方法默认直通 |
| `MetainfoPropagationMode` | `metainfo_propagation.go:30` | 前向元信息透传策略（全局默认 + 按 ctx provider 覆盖） |

## 3. 关键调用链

### 3.1 出站写元信息（客户端 WriteMeta 链）

1. `transMetaHandler.Write`（`bound/transmeta_bound.go:39`）在真正落网络前，依次调用各 `MetaHandler.WriteMeta(ctx, sendMsg)`（`:39` 内循环）。
2. `clientTTHeaderHandler.WriteMeta`（`ttheader.go:54`）把服务名/方法名/调用链等写入 `sendMsg.TransInfo().PutTransStrInfo`/`PutTransIntInfo`，随后由 TTHeader 头编解码落到线上（见 codec-payload）。
3. `metainfoClientHandler.WriteMeta`（`metainfo.go:68`）：若 ctx 含 metainfo，`metainfo.SaveMetaInfoToMap` 后 `PutTransStrInfo`（`:72`）；gRPC 路径在 `OnConnectStream`（`:44`）把 metainfo 写进 http2 metadata。

### 3.2 入站读元信息（服务端 ReadMeta 链）

1. `transMetaHandler.OnMessage`（`bound/transmeta_bound.go:51`）在业务执行前依次调用各 `ReadMeta`。
2. `metainfoServerHandler.ReadMeta`（`metainfo.go:88`）：把 `recvMsg.TransInfo().TransStrInfo()` 经 `metainfo.SetMetaInfoFromMap` 还原进 ctx，并 `WithBackwardValuesToSend`（`:92`）。
3. gRPC 服务端在 `OnReadStream`（`metainfo.go:100`）从 http2 incoming metadata 提取 metainfo 并 `TransferForward`。

### 3.3 自定义 handler 装配

- `NewCustomMetaHandler(WithWriteMeta(fn), WithReadMeta(fn), ...)`（`custom_meta_handler.go:39`）按选项注入部分方法；未注入的方法在 `WriteMeta`/`ReadMeta`（`:84/92`）中直接 `return ctx, nil` 直通，用户无需实现全部接口。

## 4. 配置项

| 配置 / 开关 | 默认 / 行为 | 位置 |
|------|------|------|
| `SetMetainfoPropagationMode` | 全局设置前向元信息透传模式 | `metainfo_propagation.go:70` |
| `SetMetainfoPropagationModeProvider` | 按 ctx 动态覆盖传播模式 | `metainfo_propagation.go:91` |
| `NewCustomMetaHandler` 选项 | WithWriteMeta/WithReadMeta/WithOnReadStream/WithOnConnectStream 按需注入 | `custom_meta_handler.go:48-73` |
| gRPC 流 LogID | `OnReadStream` 把 `stream-log-id` 写入 ctx（`addStreamIDToContext`） | `metainfo.go:117` |

## 5. 错误与重试语义

- 各 `WriteMeta`/`ReadMeta` 失败即返回 error，由 bound handler 短路中断后续管道（`transMetaHandler.Write/OnMessage` 循环遇错即返）。
- 未设置的键或空 kv 表在各 handler 中安全跳过（如 `WriteMeta` 判 `HasMetaInfo`，`ReadMeta` 判 `len(kvs)>0`）。
- gRPC 路径若 ctx 无 incoming metadata（`metadata.FromIncomingContext` 返回 false）则跳过（`metainfo.go:107`）。
- 本层无重试；透传失败属于 RPC 失败，上抛处理。

## 6. 并发细节

- 各 handler 均为无状态单例（`MetainfoClientHandler` 等 `new(...)`），方法只读 ctx/msg，无共享可变状态，天然并发安全。
- 元信息数据存于 `Message.TransInfo()`（per-message，随消息对象池化回收）与 `context.Context`，无独立 goroutine/锁。
- `MetainfoPropagationMode` 全局变量由 `atomic`/锁保护的 provider 读取（`getMetainfoPropagationMode`，`metainfo_propagation.go:103`），运行期可动态切换。
- 透传全程经 `context.Context` 传递，与 trace/超时同链。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `pkg/remote/transmeta/{metakey,http_metakey}.go` 键常量。
- `pkg/transmeta/{metainfo,ttheader,http2,metainfo_propagation}.go` 各 MetaHandler。
- `pkg/remote/custom_meta_handler.go` 与 `pkg/remote/bound/transmeta_bound.go`。

**Out-of-Scope（不在本仓库源码内）**
- metainfo 跨进程编解码与 backward/forward 值模型：`github.com/bytedance/gopkg/cloud/metainfo`（外部库）。
- http2 metadata 传输：`pkg/remote/trans/nphttp2/metadata`（见 `transport-implementations/nphttp2-grpc`）。
- TTHeader 字节格式：`gopkg/protocol/ttheader`（外部，见 codec-payload）。

## 8. 与相邻子系统交互

- **上游（trans-handler-pipeline）**：`NewTransMetaHandler([]MetaHandler)` 产物作为 `AddInboundHandler`/`AddOutboundHandler` 挂入 TransPipeline（`server/server.go`/`client/client.go` 装配）。
- **下游（codec-payload）**：本域只把 kv 写入 `message.TransInfo()`，真正落到线上由 TTHeader/HTTP2 头编解码完成。
- **横向**：metainfo handler 与业务 ctx 交互（`metainfo.SaveMetaInfoToMap`/`SetMetaInfoFromMap`）；gRPC 路径与 `nphttp2/metadata` 交互。
- **方向**：出站 ctx metainfo → WriteMeta → TransInfo/metadata → 头编码 → 网络；入站对称。

## 9. 语言专项适配口径（Go）

- **并发模型**：无 goroutine、无锁的纯函数式 handler；并发安全来自"无状态单例 + per-message/per-ctx 数据"。这是 Go 中间件/拦截器的常见模式。
- **接口与空默认实现**：`customMetaHandler` 用函数式选项 + nil 判空，提供接口的"部分实现"能力，避免用户被迫实现全部方法——Go 无接口默认方法时的惯用折中。
- **context 传播**：元信息透传完全基于 `context.Context`，与 RPC 调用链同生共死；backward 值用 `WithBackwardValuesToSend` 挂在 ctx 上随回复回传。
- **internal/边界**：`pkg/transmeta` 为公开包；`bound/transmeta_bound.go` 在 `pkg/remote/bound`，负责把 MetaHandler 适配成管道的 Inbound/Outbound handler——适配器模式隔离"元信息读写接口"与"管道事件接口"两套抽象。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|------|------|------|------|
| 元信息 handler 分层与透传架构图 | `transmeta-architecture.html` | architecture | showcase |
| 出站/入站元信息透传数据流图 | `transmeta-dataflow.html` | dataflow | standard |

JSON IR 源文件位于 `json/` 子目录。元信息透传为线性读/写流，故补 dataflow；无跨方消息交互，不单列 sequence；无状态机，不单列 lifecycle。
