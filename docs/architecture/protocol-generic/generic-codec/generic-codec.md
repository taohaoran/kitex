# 泛化调用 codec（generic-codec）

> 本文是 `protocol-generic` 域下的叶子子系统文档。域级总览见 `../protocol-generic.md`，本文只展开泛化调用的门面 `Generic` 与各 payload codec 实现，不重复展开 IDL 描述符解析与 provider（见 `../generic-descriptor/generic-descriptor.md`）与底层 Thrift 线格式（见 `../bthrift-protocol/bthrift-protocol.md`）。
>
> 源码基准：`github.com/cloudwego/kitex`，commit `4fffa48`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| 泛化调用门面接口 | `Generic` 组合 `Closer`，暴露 `PayloadCodecType/GenericMethod/IDLServiceName/GetExtra` | `pkg/generic/generic.go:34` |
| 原始 Thrift 二进制泛化 | `BinaryThriftGeneric()`（v1，已弃用）与 `BinaryThriftGenericV2(svcName)`：不依赖 IDL，直接收发原始 thrift 字节流 | `pkg/generic/generic.go:62,80` |
| 原始 Protobuf 二进制泛化 | `BinaryPbGeneric(svcName, packageName)`：直接收发 protobuf 二进制 | `pkg/generic/generic.go:100` |
| map↔Thrift 泛化 | `MapThriftGeneric(p)` / `MapThriftGenericForJSON(p)`：把 `map[string]interface{}` 动态编解码为 thrift 结构体 | `pkg/generic/generic.go:126,146` |
| HTTP↔Thrift 泛化 | `HTTPThriftGeneric(p, opts...)`：按 IDL http 注解做 HTTP 路径/查询参数到 thrift 字段的映射 | `pkg/generic/generic.go:167` |
| HTTP + PB↔Thrift 泛化 | `HTTPPbThriftGeneric(p, pbp)`：HTTP 请求同时映射 protobuf 与 thrift 描述 | `pkg/generic/generic.go:186` |
| JSON↔Thrift 泛化 | `JSONThriftGeneric(p, opts...)`：JSON 文本动态编解码为 thrift 结构体 | `pkg/generic/generic.go:207` |
| JSON↔Protobuf 泛化 | `JSONPbGeneric(pbp, opts...)`：基于 dynamicgo 做 JSON↔protobuf 转换 | `pkg/generic/generic.go:227` |
| 二进制定段选项 | `SetBinaryWithBase64` / `SetBinaryWithByteSlice` / `EnableSetFieldsForEmptyStruct` 调整二进制字段与空结构体行为 | `pkg/generic/generic.go:234,269,300` |
| 动态 thrift 结构读写器 | `thrift.StructReaderWriter` 按 `descriptor.TypeDescriptor` 逐类型分发 writer/reader 函数 | `pkg/generic/thrift/write.go:41,57`、`read.go` |
| 流式/oneway 方法信息构造 | `newMethodsMap/newMethodInfo` 按 StreamingMode 与 oneway 包装出 `serviceinfo.MethodInfo` | `pkg/generic/generic.go:626,637` |
| 泛化 Args/Result | `Args`/`Result` 类型别名（=`igeneric.Args/Result`）通过 `SetCodec(messageReaderWriter)` 携带动态读写器 | `pkg/generic/generic_service.go:211` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|-------------|------|------|
| `Generic` | `generic.go:34` | 泛化调用统一门面；每个具体泛化类型（binaryThriftGenericV2 等）实现它 |
| `DescriptorProvider` | `descriptor_provider.go:26` | 提供 `<-chan *descriptor.ServiceDescriptor` 并可 Close；IDL 热更新的推送源（实现见 descriptor 叶子） |
| `Options` / `Option` | `option.go:47` | dynamicgo 转换选项与 `useRawBodyForHTTPResp`；`DefaultHTTPDynamicGoConvOpts`/`DefaultJSONDynamicGoConvOpts` 为预置默认 |
| `mapThriftCodec` | `mapthrift_codec.go:30` | map 泛化 codec；持有 `svcDsc/readerWriter` 两个 `atomic.Value`，后台 `update()` 协程消费 provider 推送 |
| `jsonThriftCodec` | `jsonthrift_codec.go:32` | JSON 泛化 codec；含 `dynamicgoEnabled` 开关与多套 `conv.Options`（含 base/thrift 异常变体） |
| `binaryThriftCodec(V2)` / `binaryPbCodec` | `binarythrift_codec.go:40`、`binarypb_codec.go:23` | 不依赖 IDL 的原始字节透传 codec |
| `Method` / `messageReaderWriterGetter` | `generic.go:55,50` | 描述 oneway 与 StreamingMode；codec 统一通过 `getMessageReaderWriter()` 暴露动态读写器 |
| `ServiceV2` | `generic_service.go:45` | 用户实现泛化服务端时的方法集（Unary/ClientStream/ServerStream/Bidi） |

## 3. 关键调用链

1. **构建一个依赖 IDL 的泛化客户端**：用户调 `generic.MapThriftGeneric(p)`（`generic.go:126`）→ `newMapThriftCodec(p, false)`（`mapthrift_codec.go:42`）先 `svc := <-p.Provide()` 取首个 `ServiceDescriptor`，`configureMessageReaderWriter(svc)`（`mapthrift_codec.go:80`）根据 `forJSON` 选择 `thrift.NewStructReaderWriter/ForJSON`，最后 `go c.update()`（`mapthrift_codec.go:54`）启动后台协程。返回的 `Generic` 被 `genericclient.NewClient` 包装成 RPC 客户端。
2. **IDL 热更新（无锁切换）**：provider 推送新描述符 → `update()` 循环（`mapthrift_codec.go:58`）`<-c.provider.Provide()` 拿到新 `svc`，`c.svcName/svcDsc/readerWriter.Store(...)` 原子写入，再 `configureMessageReaderWriter` 重建 `*thrift.StructReaderWriter` 并 `Store`（`mapthrift_codec.go:89`）。在途请求下次 `getMessageReaderWriter()`（`mapthrift_codec.go:92`）`Load()` 即拿到新读写器，旧对象因无引用被 GC。
3. **一次泛化请求的编解码**：RPC 框架用 `newMethodInfo` 构造的 args/result 构造函数把 `getMessageReaderWriter()` 塞进 `Args.SetCodec`（`generic.go:641`）；序列化时 `thrift.write.go` 的 `typeOf`（`write.go:57`）按 thrift 类型分派到 `writeInt32/writeString/...` 等 leaf writer，配合 `descriptor.FieldDescriptor` 做字段读写；反序列化走 `read.go` 对称路径。binary 类泛化则走 `binaryThriftCodec.Marshal/Unmarshal`（`binarythrift_codec.go:44,83`）直接透传。

## 4. 配置项

| flag / option | 默认 / 行为 | 位置 |
|---------------|-------------|------|
| `MapThriftGeneric` 的 base64 | 默认关闭；`SetBinaryWithBase64(g, true)` 开启 | `generic.go:234` |
| `MapThriftGeneric` 的二进制返回类型 | 默认返回 string；`SetBinaryWithByteSlice` 改返回 `[]byte` | `generic.go:269` |
| `EnableSetFieldsForEmptyStruct` | 默认 0（不设）；1 只设 required/default，2 设全部字段（仅 map 读响应） | `generic.go:300` |
| `JSONThriftGeneric` base64 | 默认开启 | `generic.go:193` |
| `DefaultHTTPDynamicGoConvOpts` | 开启 HTTP/Value 映射、写 required/default 字段、`NoBase64Binary=true` 等 | `option.go:26` |
| `WithCustomDynamicGoConvOpts` / `UseRawBodyForHTTPResp` | 用户覆盖 dynamicgo 转换选项；控制 HTTP 响应体是否写入 `RawBody` | `option.go:66,74` |

## 5. 错误与重试语义

- `getMethod(method)` 在方法名不存在时从 `ServiceDescriptor.LookupFunctionByMethod` 返回错误，`GenericMethod()` 据此返回 `nil` `MethodInfo`（`generic.go:430`），由上层 client/server 判定为未知方法。
- `getMessageReaderWriter()` 若原子值类型不符会 `panic`（`mapthrift_codec.go:95`），属于编程错误（provider 与 codec 不匹配），不是运行时业务错误。
- `updateMessageReaderWriter()` 在 `svcDsc` 尚未就绪时返回 `"get parser ServiceDescriptor failed"`（`mapthrift_codec.go:74`）。
- 本包不实现重试；重试属 governance 域 retry 中间件（见 `../retry/retry.md`）。
- provider channel 关闭（`!ok`）时 `update()` 协程静默退出（`mapthrift_codec.go:61`），无泄漏。

## 6. 并发细节

- **goroutine 启停边界**：每个依赖 IDL 的 codec 在构造时 `go c.update()` 启动一个后台协程（`mapthrift_codec.go:54`、`jsonthrift_codec.go` 同款），该协程只从 provider channel 读，channel 关闭即退出；`Close()` 透传到 `provider.Close()`（`mapthrift_codec.go:122`）关闭上游，从而终止协程。
- **无锁热替换**：`svcDsc/svcName/combineService/readerWriter` 全部用 `sync/atomic.Value`（`mapthrift_codec.go:31,37,39`），后台协程 `Store`、请求路径 `Load`，读写无锁、无互斥；这是泛化调用能在线切换 IDL 的关键。
- **channel 通信**：`DescriptorProvider.Provide()` 返回只读 channel，fan-in 为单消费者（一个 codec 一个 `update()` 协程），非 fan-out。
- context 传递：泛化 `GenericMethod()` 按 ctx 里的 streaming mode（`igeneric.GetGenericStreamingMode(ctx)`）选择不同 `MethodInfo`（`generic.go:370`）；编解码本身不消费超时，超时由 RPC 外层传播。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `pkg/generic/`：`Generic` 门面与全部 9 种泛化工厂、`map/json/http/binary` 各 codec、`thrift/` 动态结构读写器（write.go/read.go/parse.go/struct.go 等）、`option.go`、`generic_service.go`、`streaming.go`。

**Out-of-Scope（不在本仓库源码内）**
- `github.com/cloudwego/dynamicgo`：JSON↔thrift/protobuf 高性能转换与编译加速（`conv.Options`、`dthrift.Options`），JSON/PB 泛化启用 dynamicgo 时调用。
- `github.com/cloudwego/dynamicgo/thrift`、`gopkg`：IDL 解析与底层线格式。
- `pkg/generic/descriptor/` 与 `*idl_provider*.go`：IDL 描述符结构与 provider 实现，见 `../generic-descriptor/generic-descriptor.md`。
- 上层 `genericclient`/`genericserver`（client/、server/ 包内）如何把 `Generic` 接入 RPC 调用链，不在本叶子；`remote.PayloadCodec`/`remote.ByteBuffer` 是上层 RPC 管线接口（另一分片）。
- Thrift 二进制线格式本身见 `../bthrift-protocol/bthrift-protocol.md`。

## 8. 与相邻子系统交互

- 上游 → 本叶子：用户代码（`genericclient.NewClient` / `genericserver.NewServerV2`）调用本包工厂函数拿到 `Generic`；`client/`、`server/` 把 `GenericMethod()` 产出的 `MethodInfo` 接入服务信息表。
- 本叶子 → 下游：依赖 IDL 的 codec 经 `DescriptorProvider` 拿 `*descriptor.ServiceDescriptor`（descriptor 叶子）；动态读写器在启用时调用 dynamicgo；binary 类 codec 实现 `remote.PayloadCodec` 接口供 `pkg/remote` 管线直接 Marshal/Unmarshal。
- 流向：用户选协议形态 → 工厂造 codec → codec 从 provider 拿描述符 → 按描述符建读写器 → 请求经 `Args/Result` 携带读写器走 thrift 线格式。

## 9. 语言专项适配口径

- **并发模型**：典型"单后台协程 + atomic.Value 热替换"模式——与 K8s informer 管道同构（informer 缓存 → workqueue → Reconcile → 读最新对象），区别在于 Kitex 用 channel 直推 + atomic 交换，而非 informer 索引 + workqueue 幂等重入。goroutine 生命周期严格绑定 provider channel：构造时启动、Close 关闭、`!ok` 退出，无泄漏。
- **控制器模式**：非标准 Reconcile；`update()` 是事件驱动单消费者循环，无退避、无重试（IDL 推送失败即丢弃该次更新，等待下次推送）。
- **多二进制与部署边界**：本包是被 import 的运行时库，无独立二进制；唯一二进制入口是 codegen 工具（见 `../codegen/codegen.md`）。
- **internal 边界**：`igeneric "github.com/cloudwego/kitex/internal/generic"` 定义了 `Args/Result` 与若干 extra key 常量（`generic.go:26`），体现"运行时内部类型隔离在 internal/，公开面在 pkg/generic"的依赖方向；本包不反向 import 业务层。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|----------------|
| 泛化调用 codec 分层架构图 | `generic-codec-architecture.html` | architecture | showcase |
| 泛化 codec 描述符热更新数据流 | `generic-codec-dataflow.html` | dataflow | standard（dataflow 自动布局对横边标签间距严格，缩短标签并加宽 viewBox 后通过标准档；showcase 因节点边界/标签间距未过，按流程降档，render 退出码 0） |

- 本叶子不补 sequence/lifecycle 图：泛化调用的请求时序已由系统级 RPC 时序图覆盖；codec 无独立状态机（状态仅"就绪/更新中"，且用 atomic 交换不显式建模）。
- JSON IR 源文件：`json/generic-codec-architecture.json`、`json/generic-codec-dataflow.json`。
