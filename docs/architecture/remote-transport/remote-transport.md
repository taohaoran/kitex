# remote-transport（传输核心）域总览

> 本域包含以下叶子子系统；各叶子详情见对应文档。
> 源码基准：`github.com/cloudwego/kitex`，go 1.20，commit `4fffa48`。

## 1. 域职责

本域是 Kitex 远程通信的"传输核心层"：位于 client/server 业务逻辑之下、具体网络实现之上，负责把一次 RPC 抽象为"收发处理管道（TransPipeline）+ 消息编解码（codec/payload）+ 连接管理（connpool/dialer）+ 元信息透传（transmeta）"四件事。它定义 `TransHandler`/`TransPipeline`/`Codec`/`Dialer`/`MetaHandler` 等关键抽象，使上层收发编排与底层 netpoll/gonet/http2 传输实现解耦。

核心代码路径：`pkg/remote/`（trans_handler.go、trans_pipeline.go、codec.go、payload_codec.go、connpool*.go、dialer.go、transmeta/、remotecli/、remotesvr/）。

## 2. 叶子索引

| 叶子 | 文档 | 架构图 | 时序图 / 数据流图 | 职责一句话 |
|------|------|--------|-------------------|-----------|
| trans-handler-pipeline | [trans-handler-pipeline.md](trans-handler-pipeline/trans-handler-pipeline.md) | [架构图](trans-handler-pipeline/trans-handler-pipeline-architecture.html) | [时序图](trans-handler-pipeline/trans-handler-pipeline-sequence.html) | Netty 式收发责任链 TransPipeline 与 TransHandler 接口、消息抽象 |
| codec-payload | [codec-payload.md](codec-payload/codec-payload.md) | [架构图](codec-payload/codec-payload-architecture.html) | [数据流图](codec-payload/codec-payload-dataflow.html) | 消息编解码、payload 编解码、压缩、零拷贝 ByteBuffer |
| connpool-dialer | [connpool-dialer.md](connpool-dialer/connpool-dialer.md) | [架构图](connpool-dialer/connpool-dialer-architecture.html) | [数据流图](connpool-dialer/connpool-dialer-dataflow.html) · [生命周期](connpool-dialer/connpool-dialer-lifecycle.html) | 长连接池、拨号、连接释放策略 |
| transmeta | [transmeta.md](transmeta/transmeta.md) | [架构图](transmeta/transmeta-architecture.html) | [数据流图](transmeta/transmeta-dataflow.html) | 传输元信息透传、自定义元信息 handler |

## 3. 域级机制细节

- **责任链收发**：`TransPipeline` 把 netHdlr（网络读写）、codecHdlr（编解码）、inkHdlFunc（业务）串成单向链；服务端主链 OnRead→Read→OnMessage→inkHdlFunc→Write。消息与 transInfo 用 `sync.Pool` 复用降低 GC。
- **协议自识别**：`defaultCodec` 按首字节 magic 嗅探 TTHeader/Mesh/Thrift/PB；`GetPayloadCodec` 注册表反查。
- **连接生命周期**：长连接池 FILO 取用、FIFO 驱逐；`ConnWrapper.ReleaseConn` 按错误类型/Oneway/连接重置标签决定 Put 回池、Discard 丢弃还是 Close 关闭。
- **元信息双向透传**：metainfo/ttheader/http2 各 `MetaHandler` 无状态单例，`bound/transmeta_bound.go` 把 `[]MetaHandler` 串成 `DuplexBoundHandler`，统一 WriteMeta/ReadMeta 出口。

## 4. 域级图（可选）

本域未单独产出域级架构图；各叶子架构图已覆盖组件与边界，域级关系见上方叶子索引。
