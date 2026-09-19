# 协议检测与直连 invoke（detection-invoke）

> 本文是 `transport-implementations` 域下的叶子子系统文档。域级总览见 `../transport-implementations.md`。
> 本文只展开"服务端协议检测多路分发、进程内直连 invoke、日志退避辅助"，不展开通用收发编排（见 `../remote-transport/trans-handler-pipeline/trans-handler-pipeline.md`）。
>
> 源码基准：`github.com/cloudwego/kitex`，go 1.20，commit `4fffa48`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| 协议检测多路分发 | `svrTransHandler`：包装默认 handler + 多个可检测 handler，建连后一次性嗅探协议并缓存 | `pkg/remote/trans/detection/server_handler.go:74` |
| 检测 handler 工厂 | `NewSvrTransHandlerFactory`：注入默认工厂与已注册可检测工厂 | `pkg/remote/trans/detection/server_handler.go:39` |
| noop handler | `noopHandler`：未决协议时的空实现，避免空指针 | `pkg/remote/trans/detection/noop.go` |
| 进程内 invoke handler | `ivkTransHandlerFactory`：无真实网络的直连调用 handler | `pkg/remote/trans/invoke/invoke.go:28` |
| invoke 连接扩展 | `newIvkConnExtension`：invoke 用连接扩展 | `pkg/remote/trans/invoke/conn_extension.go` |
| invoke 消息 | invoke 专用消息封装 | `pkg/remote/trans/invoke/message.go` |
| 日志指数退避 | `Exponential`：限制重复日志刷屏（初始 1s、上限 1min、5min 重置） | `pkg/remote/trans/internal/logbackoff/exponential.go:31` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|------|------|------|
| `svrTransHandlerFactory` / `svrTransHandler` | `detection/server_handler.go:48` / `:74` | 协议检测分发器；持有 `defaultHandler` 与 `registered []DetectableServerTransHandler` |
| `handlerWrapper` / `handlerKey{}` | `detection/server_handler.go:181` / `:179` | 按连接缓存"选定的真实 handler + 其 ctx" |
| `noopHandler` | `detection/noop.go` | 未决协议时的空 handler |
| `ivkTransHandlerFactory` | `invoke/invoke.go:25` | 进程内直连 handler 工厂 |
| `Exponential` | `internal/logbackoff/exponential.go:31` | 日志指数退避限频 |

## 3. 关键调用链

### 3.1 建连后一次性协议检测（OnRead）

1. `svrTransHandler.OnRead`（`detection/server_handler.go:87`）：先取 `ctx.Value(handlerKey{}).(*handlerWrapper)`（`:89`）；若 `r.handler` 已定，直接委托 `r.handler.OnRead(r.ctx, conn)`（`:91`）——复用连接不重复检测。
2. 未检测则循环 `t.registered[i].ProtocolMatch(ctx, conn)`（`:96`），首个匹配者为 `which`（`:97-99`）。
3. 匹配到则 `which.OnActive(ctx, conn)`（`:102`）；都不匹配则用 `t.defaultHandler`（`:107`）。
4. 把选定结果存入 `r.ctx, r.handler`（`:109`）缓存，再 `which.OnRead`（`:110`）。

### 3.2 方法委托

- `which(ctx)`（`:129`）：返回缓存 handler；未决时返回 `noopHandler`（`:134`）。Write/Read/OnMessage/OnInactive/OnError 全部委托给 `which(ctx)`。
- `OnActive`（`:155`）：先 `defaultHandler.OnActive`，再向 ctx 注入空 `handlerWrapper`（`:164`）占位。

### 3.3 进程内 invoke

- `newIvkTransHandler`（`invoke/invoke.go:37`）：`trans.NewDefaultSvrTransHandler(opt, newIvkConnExtension())`——复用默认收发编排，但连接扩展为进程内直连。

## 4. 配置项

| 配置 / 开关 | 默认 / 行为 | 位置 |
|------|------|------|
| 已注册检测工厂列表 | 由 `NewSvrTransHandlerFactory` 注入（如 ttheader、grpc、ttstream 等可检测 handler） | `detection/server_handler.go:39` |
| 退避常量 | initialInterval=1s、maxInterval=1min、resetAfter=5min | `internal/logbackoff/exponential.go:25-27` |

## 5. 错误与重试语义

- **协议不匹配**：所有已注册 handler `ProtocolMatch` 均失败时回退 `defaultHandler`（`server_handler.go:107`），不报错。
- **OnActive 出错**：直接返回错误中断（`:103-104/157-158`）。
- **日志退避**：`Exponential.Observe`（`exponential.go:40`）在间隔内抑制重复日志，返回累计 count/elapsed 供一次聚合输出；超过 `resetAfter` 重置间隔。

## 6. 并发细节

- **连接级缓存**：`handlerWrapper` 存于连接 ctx（`server_handler.go:164`），一连接只检测一次——避免每请求重复嗅探。
- **退避限频**：`Exponential` 用 `sync.Mutex`（`exponential.go:32`）保护 `lastLog/interval/count`。
- **context 传播**：检测出的 handler 其 `OnActive` 返回的 ctx 存入 wrapper（`:109`），后续方法用该 ctx，保证每连接独立状态。
- **goroutine**：本层不直接起 goroutine；复用下游各传输实现的并发模型。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `pkg/remote/trans/detection/` 协议检测分发；`pkg/remote/trans/invoke/` 进程内直连；`pkg/remote/trans/internal/logbackoff/` 日志退避辅助。

**Out-of-Scope（不在本仓库源码内）**
- 被检测/委托的各传输实现（ttheader、grpc、ttstream、netpoll/gonet）分别见对应叶子。
- 通用收发编排见 trans-handler-pipeline 叶子。

## 8. 与相邻子系统交互

- **上游**：服务端启动时用 detection 工厂包装各协议 handler；`remotesvr.Server` 据此统一接入多协议。
- **下游**：OnRead 按嗅探结果委托到具体传输实现（nphttp2-grpc、ttstream-mux、netpoll-trans、gonet-trans 等）。
- **横向**：invoke 用于单元测试/进程内直连调用，无真实网络。
- **方向**：统一入口 OnRead → 一次性检测 → 缓存委托 → 具体传输 handler。

## 9. 语言专项适配口径（Go）

- **委派模式 + 连接级缓存**：`svrTransHandler` 是一个"元 handler"，把多协议选择从每请求下沉为每连接一次；通过 ctx 值（`handlerKey{}`）绑定连接状态——Go 的 ctx 值传递模式。
- **internal 边界**：`pkg/remote/trans/internal/logbackoff/` 为 internal 包，仅本仓库 trans 系列可用，外部不可 import——符合 Go internal 约束。
- **无状态分发**：detection handler 本身无网络状态，状态全在被委托的下游 handler 中。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|------|------|------|------|
| 协议检测分发架构图 | `detection-invoke-architecture.html` | architecture | standard |
| 一次性协议检测时序图 | `detection-invoke-sequence.html` | sequence | showcase |

JSON IR 源文件位于 `json/` 子目录。"建连→循环检测→缓存委托"时序清晰故补 sequence；无显式业务状态机，不单列 lifecycle。
