# 高级工具（advanced-tool）域总览

> 本域包含以下叶子子系统；各叶子详情见对应文档。
> 源码基准：`github.com/cloudwego/kitex`，commit `4fffa48`。

## 1. 域职责

本域负责高级运行时装配与工程工具：xDS 动态配置与前向/反向代理、HTTP 解析；kitex 代码生成工具（唯一独立二进制）；以及被全框架复用的通用工具/预热/内存/协程封装/配置工具。

## 2. 叶子索引

| 叶子 | 文档 | 架构图 | 数据流图 | 职责一句话 |
|------|------|--------|----------|-----------|
| xds | [xds.md](xds/xds.md) | [架构图](xds/xds-architecture.html) | — | xDS 客户端套件与代理抽象 |
| codegen | [codegen.md](codegen/codegen.md) | [架构图](codegen/codegen-architecture.html) | [数据流](codegen/codegen-dataflow.html) | kitex 代码生成 CLI 工具 |
| support | [support.md](support/support.md) | [架构图](support/support-architecture.html) | — | 通用工具/预热/mem/gofunc/configutil |

## 3. 域级机制细节

- codegen 是项目唯一独立二进制（`tool/cmd/kitex`），支持 CLI 与 thriftgo/protoc 插件双模式。
- support 中的 gofunc 是框架统一 goroutine 出口（带 panic 恢复）；internal 包仅仓库内可用。

## 4. 域级图（可选）

本域不单独出域级架构图，由各叶子图覆盖。
