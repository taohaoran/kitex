# 消息编解码、payload 编解码与零拷贝缓冲（codec-payload）

> 本文是 `remote-transport` 域下的叶子子系统文档。域级总览见 `../remote-transport.md`。
> 本文只展开"协议嗅探编解码、payload 序列化注册表、压缩、零拷贝 ByteBuffer"，不展开收发管道编排（见 `../trans-handler-pipeline/trans-handler-pipeline.md`）与具体传输层字节读写（见 `transport-implementations` 域）。
>
> 源码基准：`github.com/cloudwego/kitex`，go 1.20，commit `4fffa48`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| Codec 抽象 | 整包消息编解码接口（Encode/Decode/Name），与 MetaEncoder/MetaDecoder 元信息-载荷两阶段拆分 | `pkg/remote/codec.go:24` |
| PayloadCodec 抽象 | 仅负责 payload 序列化/反序列化（Marshal/Unmarshal/Name），与协议头解耦 | `pkg/remote/payload_codec.go:29` |
| payload codec 注册表 | 进程级 `payloadCodecs map`，按 `serviceinfo.PayloadCodec` 类型查/注册 | `pkg/remote/payload_codec.go:26/36/50` |
| 默认协议嗅探编解码器 | `defaultCodec`：按首字节 magic 自动识别 TTHeader/MeshHeader/ThriftBinary/ThriftFramed/KitexProtobuf | `pkg/remote/codec/default_codec.go:100` |
| TTHeader 头编解码 | 封装 `gopkg/protocol/ttheader`，编 flags/seqID/protocolID/int+str 透传表 | `pkg/remote/codec/header_codec.go:83/97` |
| MeshHeader 解码 | 仅解码（kitex 不主动编码 mesh 头），解析 KV 透传信息 | `pkg/remote/codec/header_codec.go:192` |
| payload 校验和 | `PayloadValidator` 接口 + CRC32C 实现，发送侧生成、接收侧校验（仅 TTHeader） | `pkg/remote/codec/validate.go:42` |
| 压缩类型 | `CompressType`（NoCompress/GZip），收发压缩器名经 rpcinfo Extra 透传 | `pkg/remote/compression.go:24/33` |
| 零拷贝 ByteBuffer | Kitex 缓冲核心抽象：Next/Peek/Skip/Malloc/WriteDirect 等免拷贝读写 | `pkg/remote/bytebuf.go:48` |
| NocopyWrite/FrameWrite | 链表缓冲直写（不拷贝）、头/数据分帧写接口 | `pkg/remote/bytebuf.go:31/40` |
| Thrift payload codec | fast/frugal/apache 多模式 Thrift 序列化 | `pkg/remote/codec/thrift/thrift.go:93` |
| Protobuf payload codec | Kitex Protobuf 序列化 | `pkg/remote/codec/protobuf/protobuf.go` |
| gRPC 压缩 | grpc 子包压缩适配 | `pkg/remote/codec/grpc/grpc_compress.go` |
| 协议错误类型 | perrors 协议错误封装 | `pkg/remote/codec/perrors/protocol_error.go` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|------|------|------|
| `remote.Codec` | `codec.go:24` | 整消息编解码根接口；`Name()` 供日志 |
| `remote.MetaEncoder`/`MetaDecoder` | `codec.go:34/40` | 把编解码拆成 meta 与 payload 两阶段，供用户只覆盖其中一段 |
| `remote.PayloadCodec` | `payload_codec.go:29` | 纯 payload 序列化接口；`GetPayloadCodec` 先看 `message.PayloadCodec()` 再按 rpcinfo 配置类型查注册表 |
| `defaultCodec` | `default_codec.go:100` | 默认嗅探编解码器，内嵌 `CodecConfig{MaxSize, CRC32Check, PayloadValidator}` |
| `CodecConfig` | `default_codec.go:85` | MaxSize 限制 payload 上限；CRC32Check 开启校验和；PayloadValidator 自定义校验器 |
| `ttHeader` / `meshHeader` | `header_codec.go:81/184` | 头编解码策略；实际字节格式由外部 `gopkg/protocol/ttheader` 实现 |
| `PayloadValidator` | `validate.go:42` | Key/Generate/Validate 三方法；`getValidatorKey` 给自定义 key 加 `PV_` 前缀 |
| `remote.ByteBuffer` | `bytebuf.go:48` | 零拷贝缓冲抽象，组合 `io.ReadWriter` + 位置/长度/分帧方法 |
| `NocopyWrite` | `bytebuf.go:31` | `WriteDirect` 把 []byte 包成新节点插入链表缓冲不拷贝；`MallocAck` 修正预分配长度 |
| `ByteBufferIO` | `bytebuf.go:106` | 把 ByteBuffer 包装成标准 `io.ReadWriter`，供第三方序列化库消费 |
| `thriftCodec` | `thrift/thrift.go:93` | Thrift payload codec，按 `CodecType` 选择 fast/frugal/apache 路径 |

## 3. 关键调用链

### 3.1 发送侧编码（Encode）

1. 服务端/客户端 handler `Write` 调用 `codec.Encode`（`default_codec.go:184`）→ `EncodeMetaAndPayload`（`:152`）。
2. 若开启 `PayloadValidator` 且传输为 TTHeader，走 `encodeMetaAndPayloadWithPayloadValidator`（`:264`）：先把 payload 单独编到临时 LinkBuffer → `payloadChecksumGenerate` 算校验和 → 再写 TTHeader → `WriteDirect` 直写 payload。
3. 常规路径：`ttHeaderCodec.encode`（`header_codec.go:83`，经 `ttheader.Encode`）写头并预留 totalLenField → `me.EncodePayload`（`default_codec.go:105`）按 Framed 预分配 4 字节长度字段 → `encodePayload`（`:313`）→ `GetPayloadCodec`（`payload_codec.go:36`）→ `pCodec.Marshal`（thrift/protobuf）。
4. 回填长度：`binary.BigEndian.PutUint32(totalLenField, payloadLen)`（`default_codec.go:178`），并 `checkPayloadSize`（`:429`）校验上限。

### 3.2 接收侧解码（Decode）

1. handler `Read` 调 `DecodeMeta`（`default_codec.go:189`）：`in.Peek(2*Size32)` 偷看前 8 字节。
2. `IsTTHeader`（`:328`）/ `isMeshHeader`（`:339`）嗅探：是 TTHeader 则 `ttHeaderCodec.decode`（`header_codec.go:97`）解析头并 `PutTransIntInfo/StrInfo`；是 MeshHeader 则解 mesh 头。
3. `checkPayload`（`default_codec.go:377`）按 magic 判定 `isThriftBinary/isThriftFramedBinary/isProtobufKitex`，设置 `transProto` 与 `codecType` 到 rpcinfo 配置。
4. `DecodePayload`（`:224`）：`GetPayloadCodec` 取 codec → `pCodec.Unmarshal`；PurePayload 协议在解码后回填 `SetPayloadLen`。

### 3.3 payload codec 注册

- `internal/server/option.go:52` 与 `internal/client/option.go:60` 在初始化时 `PutPayloadCode(serviceinfo.Thrift, thrift.NewThriftCodec())` 与 `PutPayloadCode(serviceinfo.Protobuf, protobuf.NewProtobufCodec())`；`server/option.go:228/231` 支持用户自定义覆盖。

## 4. 配置项

| 配置 / 开关 | 默认 / 行为 | 位置 |
|------|------|------|
| `CodecConfig.MaxSize` | 0 = 不限 payload；>0 时超界报 `InvalidData` 协议错误 | `default_codec.go:85/429` |
| `CodecConfig.CRC32Check` | true 时自动包装 `NewCRC32PayloadValidator()` | `default_codec.go:77` |
| `CodecConfig.PayloadValidator` | 仅 TTHeader 生效；校验和值上限 4096 字节 | `validate.go:38` |
| `NewDefaultCodec` | 无大小限制的默认嗅探 codec | `default_codec.go:60` |
| Thrift `CodecType` | Basic/fast/frugal 组合，可 `DisableFastMode` 降级 | `thrift/thrift.go:37/81` |
| 压缩器 | `SetSendCompressor`/`SetRecvCompressor` 写入 rpcinfo Extra（key `send-compressor`/`recv-compressor`） | `compression.go:42/62` |

## 5. 错误与重试语义

- **协议嗅探失败**：`checkPayload` 末尾对无法识别 magic 返回 `perrors.NewProtocolErrorWithMsg("invalid payload ...")`（`default_codec.go:415`），常见于对端发来 telnet 中断消息 `0xfff4fffd`。
- **长度超限**：`checkPayloadSize` 报 `InvalidData` 类型协议错误（`default_codec.go:429`）。
- **seqID 不一致**：`ttHeader.decode` 中 `SetOrCheckSeqID` 失败仅 `klog.Warnf`（`header_codec.go:104`），不中断——部分框架头里 seqID 不可靠，真正校验在 payload 层。
- **校验和失败**：`payloadChecksumValidate` 返回 `kerrors.ErrPayloadValidation`（`validate.go:73`）。
- **编解码 panic**：由上层 handler `Read` 的 recover 兜住（见 trans-handler-pipeline 叶子），本层只返回 error。
- 本层无重试；重试由 client 侧 retry 中间件负责。

## 6. 并发细节

- **无锁注册表**：`payloadCodecs` map 在启动期一次性注册，运行期只读；`GetPayloadCodec` 并发安全（只读 map）。
- **对象复用**：`message`/`transInfo` 池化（见 trans-handler-pipeline 叶子）；编解码无独立 goroutine，完全在调用方 goroutine 同步执行。
- **零拷贝**：`NocopyWrite.WriteDirect` 把调用方 []byte 直接挂到链表缓冲节点，**不拷贝**；因此契约要求"调用方保证该 []byte 在 flush 前不被修改"（`bytebuf.go:89` WriteBinary 注释同理）。
- **context 传播**：Encode/Decode 全链路透传 `context.Context`，用于 trace 与 `stats.ChecksumGenerateStart/Finish` 打点。
- **缓冲生命周期**：`ByteBuffer.Release(e error)` 由 handler 在 defer 中按 error 释放（见 `default_server_handler.go:69/97`），flush 错误时需显式 release。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `pkg/remote/codec.go`、`payload_codec.go`、`compression.go`、`bytebuf.go` 接口。
- `pkg/remote/codec/` 下的 `default_codec.go`、`header_codec.go`、`validate.go`、`util.go`、`bytebuf_util.go` 与 `thrift/`、`protobuf/`、`grpc/`、`perrors/` 子包。

**Out-of-Scope（不在本仓库源码内）**
- TTHeader 字节格式实现：`github.com/cloudwego/gopkg/protocol/ttheader`（外部库）。
- Thrift fast/frugal 序列化底层：`frugal`、`fastpb`/` Kitex` 生成代码、`dynamicgo`（外部依赖）。
- netpoll 链表缓冲 `netpoll.LinkBuffer` 的实际零拷贝节点实现（外部库，本仓库仅 `bytebuf_util.go` 做适配封装）。
- gRPC 帧格式与 HTTP2 传输：见 `transport-implementations/nphttp2-grpc` 叶子。

## 8. 与相邻子系统交互

- **上游（trans-handler-pipeline）**：`svrTransHandler.Write/Read`（`default_server_handler.go:78/103`）持有 `remote.Codec`，在收发时调用 Encode/Decode。
- **下游（传输实现）**：handler 经 `ext.NewWriteByteBuffer/NewReadByteBuffer`（各传输实现）拿到绑定 `net.Conn` 的 `ByteBuffer`，编解码结果 flush 到连接。
- **横向**：`payload_codec.go` 注册表被 `internal/server|client/option.go` 填充；`transmeta` 的透传信息经 `message.TransInfo()` 在头编解码时读写。
- **方向**：对象 → PayloadCodec.Marshal → ByteBuffer → (TTHeader) → 网络字节流；反向对称。

## 9. 语言专项适配口径（Go）

- **并发模型**：纯同步无 goroutine；编解码是 CPU 密集型纯函数，运行在收发 goroutine 内。性能关键在于零拷贝（`NocopyWrite`/`Malloc` 预分配回填长度）与对象池复用，而非并行。
- **接口分层与依赖倒置**：`remote.Codec`（框架侧定义）依赖 `PayloadCodec`（序列化侧），具体 thrift/protobuf 实现在 `codec/{thrift,protobuf}` 子包——注册通过 `PutPayloadCode` 反向注入，避免 `remote` 包 import 具体序列化实现。
- **internal 边界**：`codec/perrors`、`codec/protobuf/error.pb.go` 为协议错误与生成代码；用户自定义 codec 通过 server/client option 注入，不破坏 `pkg/remote` 的稳定接口面。
- **Go 版本约束**：go 1.20；`sync.Pool` 用于 message/transInfo 复用；`binary.BigEndian` 处理网络序长度字段。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|------|------|------|------|
| 编解码分层与 payload 注册表架构图 | `codec-payload-architecture.html` | architecture | showcase |
| 发送/接收编解码数据流图 | `codec-payload-dataflow.html` | dataflow | standard |

JSON IR 源文件位于 `json/` 子目录。dataflow 图在 showcase 档下因回读边与写入边标签位置重叠无法通过校验，按规范降为 standard 渲染成功；调用链已由 dataflow 完整表达（对象→payload→header→字节流），不再单列 sequence；无显式状态机，不单列 lifecycle。
