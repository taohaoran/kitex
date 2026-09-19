# 协议与泛化调用（protocol-generic）域总览

> 本域包含以下叶子子系统；各叶子详情见对应文档。
> 源码基准：`github.com/cloudwego/kitex`，commit `4fffa48`。

## 1. 域职责

本域负责 RPC 线格式编解码与泛化调用能力：Thrift 二进制协议垫片、泛化调用各 codec（binary/json/http/map thrift、pb）、以及 IDL 描述符解析与 provider。这一层把"按 IDL 类型编解码"抽象成"按描述符动态编解码"，支撑不依赖生成代码的泛化调用。

## 2. 叶子索引

| 叶子 | 文档 | 架构图 | 数据流图 | 职责一句话 |
|------|------|--------|----------|-----------|
| bthrift-protocol | [bthrift-protocol.md](bthrift-protocol/bthrift-protocol.md) | [架构图](bthrift-protocol/bthrift-protocol-architecture.html) | — | Thrift 线格式二进制协议兼容垫片 |
| generic-codec | [generic-codec.md](generic-codec/generic-codec.md) | [架构图](generic-codec/generic-codec-architecture.html) | [数据流](generic-codec/generic-codec-dataflow.html) | 泛化调用各 codec 门面与热切换 |
| generic-descriptor | [generic-descriptor.md](generic-descriptor/generic-descriptor.md) | [架构图](generic-descriptor/generic-descriptor-architecture.html) | [数据流](generic-descriptor/generic-descriptor-dataflow.html) | IDL 描述符解析与 provider |

## 3. 域级机制细节

- codec 经 `DescriptorProvider.Provide()` 向 buffered channel 推送 IDL 描述符，后台 goroutine 用 atomic.Value 热切换 reader/writer，实现运行时 IDL 更新不重启。
- HTTP 泛化用基数树做路由匹配。

## 4. 域级图（可选）

本域不单独出域级架构图，由各叶子架构图覆盖。
