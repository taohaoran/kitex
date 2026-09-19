# bthrift 协议编解码（bthrift-protocol）

> 本文是 `protocol-generic` 域下的叶子子系统文档。域级总览见 `../protocol-generic.md`，本文只展开 Thrift 二进制线格式协议的编解码抽象与遗留兼容垫片，不重复展开泛化调用各 codec（见 `../generic-codec/generic-codec.md`）与 IDL 描述符解析（见 `../generic-descriptor/generic-descriptor.md`）。
>
> 源码基准：`github.com/cloudwego/kitex`，commit `4fffa48`。注意本包自带独立 go.mod（`module github.com/cloudwego/kitex/pkg/protocol/bthrift`），是 Kitex 剥离 apache/thrift 依赖迁移过程中的独立模块。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| Thrift 二进制协议读写抽象 | `BTProtocol` 接口定义消息/结构体/字段/容器/基本类型的成对读写（Read/Write）与长度预计算（XxxLength）方法 | `pkg/protocol/bthrift/interface.go:31` |
| 二进制协议实现 | 单例 `Binary`（`binaryProtocol`）实现 `BTProtocol`，逐方法委托给 gopkg | `pkg/protocol/bthrift/binary.go:28` |
| 无拷贝写扩展点 | `BinaryWriter` 类型别名 = `gopkgthrift.NocopyWriter`，配合 `WriteStringNocopy/WriteBinaryNocopy` 实现零拷贝写 | `pkg/protocol/bthrift/interface.go:28` |
| 字节/字符串分配器开关 | `SetSpanCache` 控制 gopkg 的 span cache 分配器（已弃用，转发 gopkg） | `pkg/protocol/bthrift/binary.go:36` |
| apache thrift 类型别名重导出 | `bthrift/apache` 把 apache/thrift 的 TStruct/TProtocol/TMessageType/TType、协议异常常量等别名化，供旧生成代码使用 | `pkg/protocol/bthrift/apache/thrift.go` |
| apache TStruct 互操作注册 | `apache.init()` 向 gopkg 注册 check/read/write 三个回调，使 gopkg 编解码能识别并读写 `apache.TStruct` | `pkg/protocol/bthrift/apache/apache.go:27` |
| 未知字段处理 | `UnknownField` 结构与 `GetUnknownFields/ConvertUnknownFields/UnknownFieldsLength/WriteUnknownFields`，转发 gopkg `unknownfields` | `pkg/protocol/bthrift/unknown.go:26` |
| 流式 BinaryProtocol 适配器 | `apache.BinaryProtocol` 基于 `bufiox.Reader/Writer` 实现 apache `TProtocol` 接口（Read/Write 整套方法） | `pkg/protocol/bthrift/apache/binary_protocol.go:52` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|-------------|------|------|
| `BTProtocol` | `interface.go:31` | Thrift 二进制协议读写 + 长度预计算的全集接口；实现为无状态单例，方法都是纯函数操作 `[]byte` |
| `Binary` / `binaryProtocol` | `binary.go:28` | 包级单例，`BTProtocol` 的唯一实现；本身不持有状态，仅做类型转换与转发 |
| `BinaryWriter`（=`gopkgthrift.NocopyWriter`） | `interface.go:28` | 无拷贝写回调类型，允许写入时直接持有底层 buffer 句柄避免 string→[]byte 拷贝 |
| `apache.TStruct` 等别名 | `apache/thrift.go` | 直接 `type X = thrift.X` 别名，零运行时开销，供旧生成代码 import 本包而非直接依赖 apache |
| `apache.BinaryProtocol` | `apache/binary_protocol.go:52` | 基于 bufiox 的流式 TProtocol 适配器，把字节流读写适配到 apache TProtocol 接口 |
| `UnknownField` | `unknown.go:26` | 跨包未知字段描述（Name/ID/Type/KeyType/ValType/Value 递归嵌套），用于前向兼容透传 |

## 3. 关键调用链

1. **序列化一条 Thrift 消息（写路径）**：上层 codec 拿到 `bthrift.Binary`，依次调用 `WriteMessageBegin(buf, name, typeID, seqid)`（`binary.go:40`）→ `WriteFieldBegin`（`binary.go:56`）→ `WriteI32/WriteString`（`binary.go:104/116`）→ `WriteFieldStop`（`binary.go:64`）。每个方法内部把 kitex 的 `thrift.TType/TMessageType` 强转为 gopkg 同名类型后调用 `gthrift.Binary.Xxx`，返回写入字节数；纯内存 `[]byte` 拼接，不涉及 I/O。
2. **反序列化（读路径）**：`ReadMessageBegin(buf)`（`binary.go:224`）调用 `gthrift.Binary.ReadMessageBegin` 得到 name/typeID/seqid/length，再把 `gthrift.TMessageType` 转回 `thrift.TMessageType` 返回；`ReadFieldBegin`（`binary.go:240`）、`ReadMapBegin`（`binary.go:249`）同理做类型回转。`Skip(buf, fieldType)`（`binary.go:320`）用于跳过未知/不需要的字段，递归推进读指针。
3. **apache TStruct 互操作注册（进程启动期）**：`apache.init()`（`apache.go:27`）调用 `apache.RegisterCheckTStruct(checkTStruct)`、`RegisterThriftRead(callThriftRead)`、`RegisterThriftWrite(callThriftWrite)`。当 gopkg 遇到一个实现了 `apache.TStruct` 的对象时，回调 `checkTStruct`（`apache.go:36`）做类型断言，再经 `callThriftRead`（`apache.go:44`）用 `NewBinaryProtocol` 包一层后走 `p.Read(bp)` 反序列化，`callThriftWrite`（`apache.go:55`）对称写回。

## 4. 配置项

| flag / option | 默认 / 行为 | 位置 |
|---------------|-------------|------|
| `SetSpanCache(enable bool)` | 默认行为由 gopkg 决定；控制二进制协议 bytes/string 分配器的 span cache 开关，已弃用，等价 `gthrift.SetSpanCache` | `binary.go:36` |
| 无运行时配置文件 | 本包为纯编解码函数库，无配置文件、无 flag | — |

## 5. 错误与重试语义

- 读方法（`ReadXxx`）在 buffer 长度不足、类型不合法时把 gopkg 返回的 error 直接透传给调用方（`binary.go:224-317`），本包不包装、不重试。
- `Skip` 递归跳过字段时若遇到非法类型/越界，返回 gopkg 错误（`binary.go:320`），由上层 codec 判定为协议错误并终止该消息解析。
- `checkTStruct` 对非 apache TStruct 返回固定错误 `errNotThriftTStruct`（`apache.go:35`），gopkg 据此跳过 apache 路径、走标准 fastcodec 路径。
- 本包不承担重试；重试语义属于 `governance` 域 retry 中间件（见 `../retry/retry.md`）。

## 6. 并发细节

- `binaryProtocol` 是无状态空 struct（`binaryProtocol struct{}`），所有方法都是纯函数，可被任意 goroutine 并发调用，无需加锁。
- `BTProtocol` 的读写直接操作调用方传入的 `[]byte`，不持有 buffer、不启动 goroutine、无 channel，因此无 goroutine 生命周期问题。
- `apache.init()` 在包加载时同步执行一次注册（`apache.go:27`），注册目标 gopkg 内部注册表；并发安全由 gopkg 保证（不在本仓库源码内）。
- context 不参与本包：编解码是纯内存操作，超时/取消由上层 RPC 调用链在 codec 外层处理。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `pkg/protocol/bthrift/`：`BTProtocol` 接口、`binaryProtocol` 委托实现、未知字段转换、apache 类型别名与互操作注册、流式 `BinaryProtocol` 适配器。

**Out-of-Scope（不在本仓库源码内）**
- `github.com/cloudwego/gopkg/protocol/thrift`（含 `Binary`、`NocopyWriter`、`unknownfields`、`apache` 注册中心）——真正的线格式编解码实现，本包仅做类型转换与转发。
- `github.com/apache/thrift`（TStruct/TProtocol 及协议异常常量）——被 `bthrift/apache` 别名引用，非本仓库源码。
- `github.com/cloudwego/thriftgo` 的 `extension/unknown`——未知字段 buffer 表示类型来源。
- 上层 RPC 消息帧（TTHeader 封装、长度前缀、seqid 与请求匹配）不在本包；属于 `pkg/remote` 编解码管线（另一分片）。

## 8. 与相邻子系统交互

- 上游 → 本叶子：`pkg/remote/codec/thrift` 的 Thrift 消息编解码、以及历史生成代码，通过 `bthrift.Binary` 的 Read/Write 方法做线格式序列化/反序列化。
- 本叶子 → 下游：所有实质编解码都委托 `gopkg/protocol/thrift.Binary`；apache 互操作经 `gopkg/protocol/thrift/apache` 注册中心回调。
- 与泛化 codec 关系：泛化调用的 binary/json/http/map thrift codec（见 `../generic-codec/generic-codec.md`）在做动态结构编解码时也依赖 gopkg 与 descriptor，而非直接依赖本包；本包主要服务静态生成代码的兼容路径。

## 9. 语言专项适配口径

- **并发模型**：本包是无状态纯函数库，无 goroutine、无 channel、无 mutex；唯一"全局可变状态"是包加载期 `apache.init()` 向 gopkg 注册表写入回调（`apache.go:27`），发生在任何 RPC 之前，之后只读。
- **控制器模式**：不适用——本包不是 K8s 风格的 Reconcile/informer 管道，而是无状态编解码原语库。
- **多二进制与部署边界**：本包自带独立 go.mod（见 `go.mod`），是 Kitex 为彻底剥离 `apache/thrift` 而拆出的独立模块；按 README 规划，未来 Kitex 主库不再 import 本包，仅由"含 apache thrift 代码的生成代码"依赖它。这是一个**面向迁移的兼容 shim**，不是运行时热路径。
- **internal 边界与依赖方向**：`bthrift/internal/test` 仅为测试断言工具；依赖方向为 `bthrift → gopkg → (apache/thrift)`，单向无环。`BTProtocol` 接口定义在消费侧（kitex），实现委托给 gopkg，体现"接口在本仓库、实现在外部依赖"的依赖倒置。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|----------------|
| bthrift 协议编解码架构图 | `bthrift-protocol-architecture.html` | architecture | standard（showcase 校验因连线穿节点/标签间距未过，按流程降档，render 退出码 0） |

- 本叶子不补 sequence/dataflow/lifecycle 图：本包是无状态纯函数编解码抽象，调用链为"方法委托"而非跨组件消息交互，架构图已能表达"接口→委托→外部实现"的结构；时序/管道/状态机均无独立语义。
- JSON IR 源文件：`json/bthrift-protocol-architecture.json`。
