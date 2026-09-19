# xDS 动态配置与代理（xds）

> 本文是 `advanced-tool` 域下的叶子子系统文档。域级总览见 `../advanced-tool.md`，本文只展开 `pkg/xds/`（xDS 客户端套件）、`pkg/proxy/`（前向/反向代理抽象）、`pkg/http/`（HTTP URL 解析适配），不展开具体 xDS 后端实现。
>
> 源码基准：`github.com/cloudwego/kitex`，commit `4fffa48`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| xDS 客户端套件 | `ClientSuite`：路由中间件 + Resolver | `pkg/xds/xds.go:25` |
| 套件校验 | `CheckClientSuite`：两者非 nil | `xds.go:32` |
| 前向代理接口 | `ForwardProxy.Configure/ResolveProxyInstance` | `pkg/proxy/proxy.go:41` |
| 代理配置 | `Config`：ServerInfo/Resolver/Balancer/Pool/FixedTargets | `proxy.go:31` |
| 代理中间件 | `WithMiddleware.ProxyMiddleware()` | `proxy.go:50` |
| 反向代理 | `ReverseProxy.Replace(net.Addr)` | `proxy.go:56` |
| 上下文传递 | `ContextHandler.HandleContext` | `proxy.go:66` |
| HTTP 解析 | `http.Resolver.Resolve(URL)` → 地址 | `pkg/http/resolver.go:33,71` |
| 协议族选择 | `WithIPv4/WithIPv6` | `resolver.go:40,47` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|-------------|------|------|
| `xds.ClientSuite` | `xds.go:25` | 聚合 xDS 客户端扩展 |
| `proxy.Config` | `proxy.go:31` | 发现过程所需组件 |
| `ForwardProxy` | `proxy.go:41` | 前向代理（发现+LB+连接池） |
| `ReverseProxy` | `proxy.go:56` | 反向代理（换监听地址） |
| `http.Resolver` | `resolver.go:33` | URL→地址解析 |

## 3. 关键调用链

1. **xDS 客户端装配**：启用 xDS 的 client 用 `ClientSuite{RouterMiddleware, Resolver}`（`xds.go:25`），`CheckClientSuite`（`xds.go:32`）确认非空后挂入调用链。
2. **前向代理**：`ForwardProxy.Configure(*Config)`（`proxy.go:43`）注入 Resolver/Balancer/Pool；`ResolveProxyInstance(ctx)`（`proxy.go:46`）解析出远端实例。
3. **反向代理换地址**：`ReverseProxy.Replace(net.Addr)`（`proxy.go:57`）把监听地址替换为另一个地址。
4. **HTTP 解析**：`NewDefaultResolver(...)`（`resolver.go:62`）按 WithIPv4/IPv6 配置，`Resolve(URL)`（`resolver.go:71`）把 URL 解析为 host:port。

## 4. 配置项

| flag / option | 默认 / 行为 | 位置 |
|---------------|-------------|------|
| `FixedTargets` | 逗号分隔固定 host:port，跳过发现 | `proxy.go:36` |
| `WithIPv4/WithIPv6` | 解析协议族 | `resolver.go:40,47` |

## 5. 错误与重试语义

- `Configure`/`ResolveProxyInstance` 返回 error，由 client 调用方处理。
- HTTP 解析失败返回 error（DNS 失败等）。
- 本层不做重试。

## 6. 并发细节

- 纯接口/适配层，无自有 goroutine。
- Config 在初始化期注入，运行期只读。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `pkg/xds/`、`pkg/proxy/`、`pkg/http/` 的接口与默认实现。

**Out-of-Scope（不在本仓库源码内）**
- 真正的 xDS 协议客户端（ADS/LDS/CDS）与 envoy 控制面不在本仓库；本包仅提供装配与抽象。
- 连接池见 `pkg/remote`（其他分片）；服务发现接口见 `../registry-discovery/registry-discovery.md`。

## 8. 与相邻子系统交互

- 上游 → 本叶子：client 装配层组装 ClientSuite/ForwardProxy。
- 本叶子 → 下游：依赖 discovery.Resolver、loadbalance.Loadbalancer、remote.ConnPool。
- 流向：ClientSuite → ForwardProxy → Resolver+Balancer → 连接池 → 远端。

## 9. 语言专项适配口径

- **并发模型**：无自有并发，纯装配接口。
- **控制器模式**：非 Reconcile。
- **多二进制与部署边界**：纯运行时库。
- **internal 边界与依赖方向**：`pkg/xds`/`pkg/proxy` 依赖 discovery/loadbalance/remote 等基础接口，方向向下，无环。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|----------------|
| xDS 与代理架构图 | `xds-architecture.html` | architecture | showcase |

- 本叶子不补 sequence/dataflow：纯接口装配层，无管道或调用时序。
- JSON IR 源文件：`json/xds-architecture.json`。
