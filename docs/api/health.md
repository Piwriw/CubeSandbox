# Health 类 API 原理

> 一句话概述：Health 类只暴露 `GET /health` 一个接口，用于让负载均衡 / 探针 / 运维快速判断 CubeAPI 进程「还活着并能响应 HTTP」，本身不查任何后端、不做鉴权、不限流。

> 同类文档：[cluster.md](./cluster.md) · [sandboxes.md](./sandboxes.md) · [templates.md](./templates.md)

## 接口清单

| 方法 | 路径 | operationId | 简介 | 鉴权 |
|------|------|-------------|------|------|
| GET  | `/cubeapi/v1/health`（`openapi.yml` 中相对路径为 `/health`） | `health` | 返回服务存活状态与（占位的）沙箱计数 | 无 |

补充：同一个 handler 还额外挂在根域 `GET /health`（E2B 兼容前缀），见下文「关键代码引用」中 `routes.rs` 的两处挂载点。

## 1. GET /health

### 接口定义

- 方法 / 路径：`GET /health`（在 `openapi.yml:59-71` 定义；线上通过 `routes.rs:49` 的 `.nest("/cubeapi/v1", ...)` 前缀化，最终对外路径为 `GET /cubeapi/v1/health`）。
- 查询参数：无。
- 请求体：无。
- `operationId`：`health`，`tags: handlers::health`。
- 响应（仅声明 `200`，无其他错误码）：

| 字段 | 类型 | 必填 | 含义 |
|------|------|------|------|
| `status` | `string` | 是 | 固定为字面量 `"ok"`（`health.rs:38`） |
| `sandboxes` | `integer`，`minimum: 0` | 是 | 当前固定返回 `0`（`health.rs:38`，占位字段，详见下文「注意事项」） |

Schema 来源：`openapi.yml:586-596`（`HealthResponse`）。Rust 侧的对应类型在 `health.rs:13-17` 用 `#[derive(Serialize, ToSchema)]` 同名声明，两份 schema 由 `utoipa` 在编译期对齐。

### 数据流

```
HTTP GET /cubeapi/v1/health
        │
        ▼
axum Router (routes.rs:62-65 合并后的顶层 Router)
        │
        ├── apply_http_layers (routes.rs:354-363)
        │     SetRequestId → TraceLayer → TimeoutLayer(30s) → Compression → CORS
        │     注意：这一层是「全局 HTTP 中间件」，对 /health 也生效；
        │           但鉴权 / 限流中间件是「按路由子树挂载」的，
        │           /health 没有走进 with_auth / with_auth_and_rate_limit。
        │
        ▼
nest("/cubeapi/v1", build_cubeapi_router)  (routes.rs:49, 83-91)
        │
        ▼
.route("/health", get(health::health))     (routes.rs:85)
        │
        ▼
health::health(State<AppState>)            (health.rs:27)
        │
        ├── tracing::debug!("health: ok")            (health.rs:28)
        ├── state.logger.log(LogEvent::new(Debug, "api.request")
        │       .field("handler", "health"))         (health.rs:29-32)
        │     → FilteredLogger(MinLevel=Info) 会过滤掉这条 Debug 事件，
        │       除非启动时带 --debug（见 main.rs:139-141、205-209）
        │
        └── 返回 (StatusCode::OK, Json(HealthResponse { status:"ok", sandboxes:0 }))
                                                  (health.rs:34-40)
        │
        ▼
HTTP 200  {"status":"ok","sandboxes":0}
```

关键点：handler 内部**没有调用任何 service、没有访问 CubeMaster、没有读 `state.rate_limiter` / `state.services.sandboxes`**。`State(state)` 只是为了拿到 `state.logger` 写一条结构化日志（`health.rs:27-32`）。

### 实现细节

1. **state 的来源**：`AppState` 由 `main.rs:231` 在启动时构造（`state::AppState::new(cfg, logger)`），随后通过 `routes::build_router(state)` → `Router::with_state(state)`（`routes.rs:65`）注入到 axum，请求时由 axum 的 `State` extractor 克隆传递给 handler（`state.rs:13-15` 注明 Axum 每个请求都会 clone，所以全部字段都是 `Arc`/可廉价克隆的）。

2. **「sandbox 计数」怎么拿**：当前是硬编码的 `0`（`health.rs:38`）。`HealthResponse` 结构体（`health.rs:13-17`）虽然定义了 `sandboxes: usize` 字段，但 handler 并未从 `state.services.sandboxes` 或任何 `CubeMasterClient` 接口取真实计数。也就是说 OpenAPI schema 里的 `sandboxes` 是一个**契约层面的占位字段**，等价于「还没接」。

3. **是否有缓存**：无。每次请求都直接构造一个新的 `HealthResponse`，无 TTL、无 memoize、无共享计数器。

4. **日志路径**：handler 同时写两份日志——
   - 一条 `tracing::debug!("health: ok")`（`health.rs:28`）走 stdout 订阅者（`main.rs:175-178`）；
   - 一条结构化 `LogEvent{ level=Debug, event="api.request", fields={handler:"health"} }`（`health.rs:29-32`）走 `state.logger` → `FilteredLogger` → `MultiLogger` → `FileLogger`（`main.rs:211-221`）。由于默认 `min_level = Info`（`main.rs:205-209`），非 `--debug` 启动时这两条 Debug 日志都不会落盘。

5. **错误处理**：handler 全是同步确定的字面量，**没有 `Result` 返回，不会产生 5xx**。理论上唯一可能触发的非 200 是全局 `TimeoutLayer`（`routes.rs:26`、`359`，30 秒）命中——但 handler 本身不阻塞，不可能超时。

### 关键代码引用

- `CubeAPI/src/handlers/health.rs:13-17` — `HealthResponse` 结构体（`status` / `sandboxes` 两字段，与 OpenAPI schema 一致）。
- `CubeAPI/src/handlers/health.rs:20-26` — `#[utoipa::path(...)]` 注解，把该接口注册进生成的 OpenAPI 文档。
- `CubeAPI/src/handlers/health.rs:27-41` — `health` handler 主体：拿 `State`、写两条 Debug 日志、返回固定 200 + JSON。
- `CubeAPI/src/handlers/mod.rs:9` — `pub mod health;`，把 handler 模块暴露给上层。
- `CubeAPI/src/routes.rs:70` — 把 `/health` 挂在**根路径**（E2B 兼容前缀，`build_e2b_router`）。
- `CubeAPI/src/routes.rs:85` — 把 `/health` 挂在 `/cubeapi/v1` 子树下（`build_cubeapi_router`，即 OpenAPI 契约里的对外路径）。
- `CubeAPI/src/routes.rs:83-91` 与 `with_auth` / `with_auth_and_rate_limit` 的对比 — `/health` 是直接 `.route(...)`，**未经过** `unified_auth` 和 `rate_limit`，因此任何客户端不带凭证也能命中。
- `CubeAPI/src/main.rs:60-63` — CLI 帮助文本里明确写「every API request (except GET /health) must carry either Authorization: Bearer ... or X-API-Key ...」，印证了 /health 的免鉴权是设计意图而非疏漏。
- `CubeAPI/src/state.rs:17-35` — `AppState` 结构定义，handler 仅用到其中的 `logger` 字段。
- `CubeAPI/src/logging/mod.rs:42-47、65-83、86-90` — `LogLevel::Debug`、`LogEvent::new(...)`、`LogEvent::field(...)` 的签名，对应 `health.rs:29-32` 的链式调用。
- `openapi.yml:59-71` — `/health` 路径定义；`openapi.yml:586-596` — `HealthResponse` schema（`status: string`、`sandboxes: integer ≥ 0`）。

### 错误码

| HTTP | 触发条件 | 说明 |
|------|----------|------|
| 200  | 正常命中 handler（`health.rs:35`） | 唯一会产生的业务响应；body 始终为 `{"status":"ok","sandboxes":0}` |
| 408  | 全局 `TimeoutLayer`（30 s，`routes.rs:359`）触发 | 仅当进程被事件循环卡死 30 s 才可能命中，handler 自身不会触发 |
| — 401 / 429 | **不会** | `/health` 未挂 `unified_auth` / `rate_limit` 中间件 |
| — 5xx | **不会** | handler 无 `Result` 返回，无内部调用，不产生 `AppError` |

### 注意事项 / 设计意图

1. **`sandboxes` 字段是死值**：handler 写死 `sandboxes: 0`（`health.rs:38`）。schema 里保留它是为了未来对接 `state.services.sandboxes` 真实计数而不破坏契约，但目前**不要把 0 当成「集群里真的没有沙箱」**。如果要做存活+容量探针，应另走 `GET /cluster/overview`（见 `openapi.yml:17-40`、`routes.rs:255`）。
2. **故意不做鉴权 / 限流**：路由在 `build_cubeapi_router`（`routes.rs:83-91`）里是裸 `.route`，与 `with_auth`（`routes.rs:328-338`）/ `with_auth_and_rate_limit`（`routes.rs:340-352`）的包裹分支完全无关。`main.rs:60-62` 的 CLI 文档明确点名 `/health` 是鉴权的唯一例外。设计动机：探针/负载均衡器不携带凭证，必须永远能探到。**代价**：`/health` 暴露了进程存活信息，若部署在公网需要前置网关。
3. **双路径挂载**：同一个 `health::health` 同时挂在 `/health`（E2B 兼容）和 `/cubeapi/v1/health`（OpenAPI 对外）。两份契约共享同一 handler，行为完全等价。`routes.rs:384-419` 的测试 `preserves_root_e2b_routes` / `serves_web_routes_under_cubeapi_prefix` 锁定了这一双路径行为。
4. **日志可能「沉默」**：handler 默认只发 Debug 级日志（`health.rs:28、31`），而生产默认 `min_level = Info`（`main.rs:208`）。如果不带 `--debug` 启动，访问 `/health` 在结构化日志文件里**看不到任何记录**，只有 tracing 的 stdout 订阅者可能受 `LOG_LEVEL` 影响。运维排查时若发现 `/health` 访问没落日志，先确认启动参数与 `LOG_LEVEL`，不要误判为「没被调用」。
5. **没有 TODO 标记**：`health.rs` 与路由侧均无 `TODO` / `FIXME` 注释；`sandboxes` 的真实取值是一个**未在源码里显式标注**的待办，需要后续接入 service 层才能落地。
