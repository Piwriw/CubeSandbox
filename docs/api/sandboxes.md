# Sandboxes 类 API 原理

> 这类 API 负责沙箱生命周期管理（查询、删除、暂停、恢复、列表、日志）。**创建**沙箱（`POST /sandboxes`）虽然在 CubeAPI 内部也实现了 handler，但 SDK 直连 CubeMaster 创建路径并不经过本网关，本文档不把它列入「sandboxes 类对外契约」。SDK 走 SDK→CubeMaster 直连；本类接口走 SDK→CubeAPI→CubeMaster。

> 同类文档：[health.md](./health.md) · [cluster.md](./cluster.md) · [templates.md](./templates.md)

涉及源码（除 `openapi.yml` 外，所有 `file:line` 均在 `CubeAPI/` 下）：

- `src/handlers/sandboxes.rs`、`src/handlers/mod.rs`
- `src/services/sandboxes.rs`、`src/services/mod.rs`
- `src/cubemaster/mod.rs`、`src/middleware/auth.rs`
- `src/models/mod.rs`、`src/error/mod.rs`、`src/routes.rs`
- `openapi.yml`（sandboxes path 与 schema）

---

## 接口清单

| 方法 | 路径 | operationId | 简介 | 鉴权 |
|---|---|---|---|---|
| GET | `/sandboxes/{sandboxID}` | `get_sandbox` | 取单个沙箱详情（404 不存在） | unified_auth + rate_limit |
| DELETE | `/sandboxes/{sandboxID}` | `kill_sandbox` | 同步销毁沙箱（404 / 409 paused） | unified_auth + rate_limit |
| POST | `/sandboxes/{sandboxID}/pause` | `pause_sandbox` | 暂停沙箱（404 / 409） | unified_auth + rate_limit |
| POST | `/sandboxes/{sandboxID}/resume` | `resume_sandbox` | 恢复沙箱，返回 201（404 / 409） | unified_auth + rate_limit |
| GET | `/v2/sandboxes` | `list_sandboxes_v2` | 列表，支持 metadata / state / limit | unified_auth + rate_limit |
| GET | `/v2/sandboxes/{sandboxID}/logs` | `get_sandbox_logs_v2` | 结构化日志（cursor + limit） | unified_auth + rate_limit |

> 4xx 摘要列在「简介」括号里；完整错误码见共性设计章节。鉴权机制见共性设计章节。

路由注册见 `src/routes.rs:116-154`（`build_sandbox_routes`）。鉴权 + 限流分层见下文「共性设计」。

> 提醒：`POST /sandboxes`、`GET /sandboxes`（v1）、`POST /sandboxes/{id}/connect`、`POST .../timeout`、`POST .../refreshes`、`GET .../logs`（v1）这些路由也在 `build_sandbox_routes` 中注册（`src/routes.rs:118-138`），但本次「sandboxes 类」对外文档集只覆盖上面 6 个。v1 logs / v1 list 同样保留在路由中，新集成建议直接使用 v2。

---

## 状态机

### 三态定义

CubeAPI 对外暴露的 `SandboxState` 枚举（`src/models/mod.rs:34-40`、`openapi.yml:927-933`）只有三态：

```
running | paused | pausing
```

但 CubeMaster 内部的容器状态码更细，CubeAPI 在反序列化时把它折叠到这三态上（`src/cubemaster/mod.rs:1168-1204`）：

| CubeMaster 数字码 | 语义 | CubeAPI 折叠为 |
|---|---|---|
| 1 | CONTAINER_RUNNING | `running` |
| 4 | CONTAINER_PAUSING | `pausing` |
| 5 | CONTAINER_PAUSED | `paused` |
| 2 | CONTAINER_EXITED | 经 `normalize_sandbox_status_text` 规范化为 `"stopped"`（`cubemaster/mod.rs:1192-1204`），再被 `sandbox_state_from_str` 折成 `running` |
| 0/3/其它 | CREATED / UNKNOWN | `running`（默认值） |

折叠分两条路径：

- **详情路径**（`get_sandbox`）走 `SandboxStatus` 枚举（`src/cubemaster/mod.rs:597-607`，按数字解码，见 `into_first_sandbox` `src/cubemaster/mod.rs:1233-1241`），最后再用 `sandbox_state_from_status`（`src/services/sandboxes.rs:789-795`）压回 `Running` / `Paused`，**`pausing` 在这条路径上会被压成 `running`**。
- **列表路径**（`list_sandboxes_v2`）走 `status` 字符串字段（`src/cubemaster/mod.rs:979-1002`，`deserialize_sandbox_status` `src/cubemaster/mod.rs:1180-1192`），字符串 "pausing" 能保留下来，再由 `sandbox_state_from_str`（`src/services/sandboxes.rs:797-803`）映射为 `Pausing`。因此 `pausing` 状态**只在列表 API 出现**，单查详情时对外表现为 `running`。

### 状态转换图

```
                create (SDK→CubeMaster 直连)
                          │
                          ▼
                       ┌───────┐
        POST /pause    │       │   POST /resume
     ┌──────────────►  │ running│  ◄─────────────┐
     │                 │       │                 │
     │                 └───┬───┘                 │
     │                     │                     │
     │              DELETE │ (409 if paused)     │
     │                     ▼                     │
     │                  ┌───────┐                │
     │      POST /pause │       │ POST /resume   │
     │   ┌─────────────►│ paused│────────────────┘
     │   │  (409: cannot│       │  (409 if already
     │   │   be paused) └───────┘   running or capacity)
     │   │
     │ ┌─┴──────────────┐
     │ │ POST /resume   │   pausing 仅在 list 路径可见
     └►│  on paused     │   单查 get_sandbox 时被压成 running
       └────────────────┘
```

### 409 触发条件（逐接口）

- `pause_sandbox`：`POST /cube/sandbox/update` action=`pause`，CubeMaster 返回 `ret_code = 130409`（`RET_CODE_CONFLICT`，`src/services/sandboxes.rs:28`）。注意：`parse_response`（`src/cubemaster/mod.rs:1620-1624`）在业务码非 0/200 时**直接抛 `CubeMasterError::Api`**，所以业务码错走 `.map_err(map_update_cubemaster_err)`（`src/services/sandboxes.rs:278`）路径，**不会**进入后面的 `ensure_update_result`。无 `ret_msg` 时回退文案是 `"sandbox {id} conflict"`（`src/services/sandboxes.rs:672-681`）。HTTP 409 的样例语义是「沙箱不在 running 状态（如已 paused / pausing / exited）」。`ensure_update_result` 的 `conflict_message = "cannot be paused"` 入参（`src/services/sandboxes.rs:280-285`）实际上**永远不会被用到**——它是死代码（见下文「死代码」）。
- `kill_sandbox`：默认走 `DELETE /cube/sandbox`，**404 才是常规错误**；但若 CubeMaster 返回 `130593`（`RET_CODE_MASTER_INTERNAL`，`src/services/sandboxes.rs:29`），CubeAPI 会再发一次 `GET /cube/sandbox/info` 探活，若探到的是 `Paused`，则**人为合成 409** `"sandbox {id} is paused; resume it before deleting"`（`src/services/sandboxes.rs:247-264`）。这是 paused 沙箱删除被拒的唯一显式信号；非 paused 的内部错误继续走 500。

---

## 1. `GET /sandboxes/{sandboxID}`

### 接口定义

- 鉴权：通用 `unified_auth`（见共性设计）。
- 返回 200 `SandboxDetail`（`openapi.yml:832-894`）；404 `ApiError`；500 `ApiError`。
- path 参数 `sandboxID`：字符串，不经 `validate_path_segment`（那个仅用于 snapshot/template 路径，本接口直传给 CubeMaster）。

### 数据流

```
HTTP GET /sandboxes/{id}
   │
   ▼  src/handlers/sandboxes.rs:132-155 (get_sandbox)
   │  → 打点 LogEvent("api.request")
   ▼
SandboxService::get_sandbox  src/services/sandboxes.rs:113-147
   │  ① fetch_sandbox_detail(id)  → GET /cube/sandbox/info?sandbox_id=&instance_type=
   │       CubeMasterClient::get_sandbox  src/cubemaster/mod.rs:112-126
   │       parse_response → GetSandboxResponse → into_first_sandbox
   │  ② fetch_sandbox_summary(id, host_id) → POST /cube/sandbox/list (host_id 过滤)
   │       CubeMasterClient::list_sandboxes  src/cubemaster/mod.rs:92-105
   │  ③ 组装 SandboxDetail
   ▼
Json(SandboxDetail)  → 200
```

### 实现细节

- **两次 RPC 合并字段**：详情 RPC 返回的是 `GetSandboxDataItem`，**不带 `started_at`**（只有 container 的 `create_at` 纳秒），列表 RPC 才直接给 `started_at`。CubeAPI 用 summary 优先、detail 兜底的策略组装（`src/services/sandboxes.rs:115-127`）。
- **`endAt` 的「never-timeout」语义**：CubeMaster 对永不过期沙箱不返回 `endAt`。注释明确「不要把它折叠到 `startedAt` 上，否则会被解读成『已过期』」(`src/services/sandboxes.rs:121-127`)。落到 `SandboxDetail` 上时为 `Option<DateTime<Utc>>`，`None` 在序列化时省略（`src/models/mod.rs:296-299`）。
- **`state` 来源**：`into_first_sandbox` 把数字状态码映射到 `SandboxStatus`（`src/cubemaster/mod.rs:1233-1241`），再由 `sandbox_state_from_status` 压成 `Running` / `Paused`（`src/services/sandboxes.rs:789-795`）。即详情路径**不会**返回 `pausing`。
- **metadata / volumeMounts 来源**：metadata 来自 CubeMaster `labels`（`src/services/sandboxes.rs:143` 调 `optional_metadata(d.labels)`，空 map 转 `None`）；`volumeMounts` 始终填 `None`（同上）——该字段是历史兼容位，详情路径不返回。`envd_version` 来自 annotation `ENVD_VERSION_ANNOTATION`，缺失回退到 `ENVD_VERSION_FALLBACK`（`envd_version_from_annotations` `src/services/sandboxes.rs:716-723`）。
- **CPU / 内存解析**：详情 RPC 给的是 container 维度的字符串（如 `"1000m"`、`"2048Mi"`），由 `parse_cpu_millicores` / `parse_mem_mb` 解析（`src/cubemaster/mod.rs:1081-1093`）。`disk_size_mb` 在详情路径写死 `0`（`src/cubemaster/mod.rs:1254`）。
- **404 路径**：CubeMaster HTTP 404、业务码 130404、或返回数据为空，统一 `AppError::NotFound`（`src/services/sandboxes.rs:484-504`）。

### 关键代码引用

- handler：`src/handlers/sandboxes.rs:132-155`
- service：`src/services/sandboxes.rs:113-147`
- CubeMaster 详情：`src/cubemaster/mod.rs:112-126`、`1221-1258`
- 状态折叠：`src/cubemaster/mod.rs:1168-1204`、`src/services/sandboxes.rs:789-795`

---

## 2. `DELETE /sandboxes/{sandboxID}`

### 接口定义

- 返回 204（无 body）；404 / 500 `ApiError`（`openapi.yml:160-185`）。openapi 没显式声明 409，但代码路径会合成（见下）。
- **同步**销毁：`DeleteSandboxRequest.sync = Some(true)`（`src/services/sandboxes.rs:237-245`）——CubeAPI 等待 CubeMaster 把底层资源（rootfs、网络、cubelet 元数据）真正释放后才返回 204。

### 数据流

```
HTTP DELETE /sandboxes/{id}
   ▼  handler kill_sandbox  src/handlers/sandboxes.rs:205-226
SandboxService::kill_sandbox  src/services/sandboxes.rs:237-271
   │  DELETE /cube/sandbox { sync: true }
   │    CubeMasterClient::delete_sandbox  src/cubemaster/mod.rs:76-89
   ▼
   ├─ 成功：ret_code 0/200 → Ok(()) → 204
   ├─ 130404（not found）：→ AppError::NotFound → 404
   └─ 130593（master internal）：再 GET /cube/sandbox/info 探活
        ├─ 当前 status == Paused → 409 "is paused; resume it before deleting"
        └─ 其它 / 探活失败 → 500
```

### 实现细节

- **同步等待 + 超时**：本路由走标准 30s 路由超时（`src/routes.rs:26、46-66、116-154`），**不在** `snapshot_long_router` 上。若 CubeMaster 的同步删除超过 30s（极少见，paused→resume→delete 路径才会触发），客户端会收到 408 而底层删除仍在进行。
- **paused 删除的二次探活**：`RET_CODE_MASTER_INTERNAL` 是 CubeMaster 删除路径常见的「业务层错误」，但其中相当一部分是「沙箱处于 paused、Cubelet 拒绝销毁」这类语义。CubeAPI 选择**再发一次 info 请求**做精细分类（`src/services/sandboxes.rs:247-264`）：
  - 探到 `SandboxStatus::Paused` → 合成 409，提示调用方先 resume；
  - 其它（已删除、网络抖动、info 也失败）→ 保持 500。
- **没有重试**：`CubeMasterClient` 是 `reqwest::Client` 的薄封装，**不带 backoff / retry**（`src/cubemaster/mod.rs:43-55`，全文件 grep 无 retry/backoff）。一次失败即失败。
- **资源回收**：rootfs、网络命名式、cubelet 侧元数据都在 CubeMaster 的同步删除路径内完成；CubeAPI 自身不持有任何外部资源（无 DB 行、无临时文件需要清理）。

### 关键代码引用

- handler：`src/handlers/sandboxes.rs:205-226`
- service：`src/services/sandboxes.rs:237-271`
- 探活逻辑：`src/services/sandboxes.rs:247-264`、`fetch_sandbox_detail` `src/services/sandboxes.rs:476-504`
- CubeMaster 删除：`src/cubemaster/mod.rs:76-89`

---

## 3. `POST /sandboxes/{sandboxID}/pause`

### 接口定义

- 返回 204（无 body）；404 / 409（`"Sandbox cannot be paused"`） / 500（`openapi.yml:186-218`）。
- 无 request body。

### 数据流

```
HTTP POST /sandboxes/{id}/pause
   ▼  handler pause_sandbox  src/handlers/sandboxes.rs:243-264
SandboxService::pause_sandbox  src/services/sandboxes.rs:273-286
   │  build_update_request(id, "pause", None)
   │  POST /cube/sandbox/update  src/cubemaster/mod.rs:129-142
   ▼
   ├─ 0/200  → 204
   ├─ 130404 → 404 "sandbox {id} not found"
   └─ 130409 → 409 "cannot be paused"（或回传 CubeMaster 原因）
```

### 实现细节

- **机制是 container-level freeze，不是 VM-level suspend**：CubeMaster 的 update action=`pause` 操作的是容器 cgroup freezer（状态码语义见 `src/cubemaster/mod.rs:1168-1170` 的注释 `0=CREATED, 1=RUNNING, 2=EXITED/STOPPED, 3=UNKNOWN, 4=PAUSING, 5=PAUSED`，以及 `sandbox_status_text_from_code` `:1167-1180` 的数字→字符串映射），而不是整 VM 挂起。memory 状态保留，恢复延迟极低。
- **请求体**：`SandboxUpdateRequest`（`src/cubemaster/mod.rs:1270-1284`）只携带 `requestID / sandbox_id / instance_type / action="pause"`，**不带 timeout**（pause 也不需要新 TTL）。
- **`pausing` 中间态怎么暴露**：CubeMaster 在真正完成 freeze 前会先置 status=4（`pausing`）。但本接口是**同步**的——`ret_code=0` 返回时沙箱已经是 paused。`pausing` 只在 client 并发查询列表（`GET /v2/sandboxes`）或多次 pause 重复触发时才会被读到，且仅在列表路径上才能映射为 `Pausing`（详情路径会被压成 running，见状态机小节）。
- **409 触发**：业务码 130409 经 `parse_response` 抛出为 `CubeMasterError::Api`（`cubemaster/mod.rs:1620-1624`），由 `map_update_cubemaster_err`（`services/sandboxes.rs:670-682`）翻译为 `AppError::Conflict`，保留 CubeMaster 的 `ret_msg`（若有），否则回退到 `"sandbox {id} conflict"`。
- **可重复性**：对 paused 沙箱再次 pause 也会走 409，由 CubeMaster 拒绝；CubeAPI 不做幂等化。

### 关键代码引用

- handler：`src/handlers/sandboxes.rs:243-264`
- service：`src/services/sandboxes.rs:273-286`
- update RPC：`src/cubemaster/mod.rs:129-142`、`1267-1289`
- 错误映射：`src/services/sandboxes.rs:664-711`

---

## 4. `POST /sandboxes/{sandboxID}/resume`

### 接口定义

- request body：`ResumedSandbox`（`openapi.yml:788-796`、`src/models/mod.rs:321-330`）。
  ```yaml
  ResumedSandbox:
    description: Request body for POST /sandboxes/{id}/resume (deprecated).
    properties:
      autoPause: { type: boolean }
      timeout:   { type: integer, format: int32 }
  ```
  字段含义：
  - `timeout`：恢复后的 idle TTL（秒）。`None` 表示不主动覆盖（透传给 CubeMaster，由 master 保留原 TTL）。`0` 在 update 协议里语义为「保持原 timeout」（见 `SandboxUpdateRequest.timeout` 注释，`src/cubemaster/mod.rs:1281-1283`），并不是立即过期。
  - `autoPause`：**被忽略**——`#[allow(dead_code)]`（`src/models/mod.rs:322-323`）。从 handler 到 service 全程没读这个字段。
- 返回 **201** `Sandbox`（注意不是 `SandboxDetail`，少了 state / metadata / 资源量，但多了 `envdAccessToken`、`trafficAccessToken` 等访问凭据位）。
- 错误：404 / 409（`"Sandbox is already running"`）/ 500（`openapi.yml:219-261`）。

### 数据流

```
HTTP POST /sandboxes/{id}/resume  (body: ResumedSandbox)
   ▼  handler resume_sandbox  src/handlers/sandboxes.rs:282-310
SandboxService::resume_sandbox(id, body.timeout)  src/services/sandboxes.rs:288-319
   │  ① build_update_request(id, "resume", timeout)
   │     POST /cube/sandbox/update
   │  ② ensure_update_result（404 / 409 / 500）
   │  ③ fetch_sandbox_detail(id)（resume 后再拉一次详情，填 template_id/host_id）
   │  ④ sandbox_response(...)（traffic_access_token 永远为 None）
   ▼
201 Created + Json(Sandbox)
```

### 实现细节

- **`autoPause` 是死字段，deprecated 来源**：`openapi.yml` 把 `ResumedSandbox` 标为 `(deprecated)`，`src/models/mod.rs:321` 的 doc comment 也写「Request body for POST /sandboxes/{id}/resume (deprecated)」。deprecated 的来由是 SDK 历史上把 `lifecycle.autoPause` 暴露成 resume 时的开关，但 CubeAPI 已经把生命周期决策全部归到 `create_sandbox` 的 `lifecycle` 对象里（见 `services/sandboxes.rs:185-197`），resume 时不再允许修改该位。字段保留是为了不破坏老 SDK 的请求体。
- **`trafficAccessToken` 永远为 `None`**：注释明确「token 只在 create 时返回给 caller 持久化，之后 CubeProxy 直接从 Redis 读；resume/connect 路径调 `fetch_sandbox_detail`，该 RPC 不暴露 token，因此这里 None 是正确的」（`src/services/sandboxes.rs:308-318`）。空字符串同样被过滤掉（`sandbox_response` `src/services/sandboxes.rs:546-558`）。
- **201 vs 409**：成功路径**显式返回 `StatusCode::CREATED`**（`src/handlers/sandboxes.rs:309`）——语义是「恢复 = 新建一次 running 实例」。409 由 `map_update_cubemaster_err` 在业务码 130409 时抛出（路径同 pause：`parse_response` 抛出 → `.map_err` 翻译），冲突文案优先用 CubeMaster 给的 `ret_msg`，否则回退到 `"sandbox {id} conflict"`。
- **resume 容量拒绝**：`map_update_cubemaster_err` 会把 CubeMaster 给出的 `ret_msg` 原文回传，便于调用方看到具体拒绝原因。注释里点出的 `paused_resource_release_ratio` 水位拒绝属于这一类（注释见 `src/services/sandboxes.rs:699-703`，但实际触发文案回退走的是 `map_update_cubemaster_err`，不是 `ensure_update_result`）。调用方应据此降级。

### 关键代码引用

- handler：`src/handlers/sandboxes.rs:282-310`
- service：`src/services/sandboxes.rs:288-319`
- `ResumedSandbox` 模型：`src/models/mod.rs:321-330`
- `SandboxUpdateRequest`：`src/cubemaster/mod.rs:1270-1284`

---

## 5. `GET /v2/sandboxes`

### 接口定义

- query 参数（`ListSandboxesV2Query`，`src/models/mod.rs:519-534`）：
  - `metadata`：`string?`，`key1=val1&key2=val2` 形式的 metadata 过滤（见下）。
  - `state`：`string?`，仅识别 `"running"` / `"paused"`，其它值（含 `"pausing"`）一律视为不过滤（`parse_state_filter` `src/services/sandboxes.rs:777-783`）。
  - `nextToken`：`string?`，**保留字段，未实现**。
  - `limit`：`int32`，默认 100（`default_page_limit`，`src/models/mod.rs:532-534`），**无服务端硬上限**。
- 返回 200 + `ListedSandbox[]`（`openapi.yml:597-651`）；只有 500 一种错误响应。

### 数据流

```
HTTP GET /v2/sandboxes?metadata=...&state=...&limit=...
   ▼  handler list_sandboxes_v2  src/handlers/sandboxes.rs:83-116
SandboxService::list(metadata_filter, state_filter, limit)  src/services/sandboxes.rs:80-111
   │  POST /cube/sandbox/list { start_idx: 0, size: max(limit,1) }
   │    CubeMasterClient::list_sandboxes  src/cubemaster/mod.rs:92-105
   │  → ListSandboxResponse.sandboxes: Vec<SandboxInfo>
   ▼
   逐条 from_cubemaster_info（src/services/sandboxes.rs:725-753）
   ├─ filter_by_metadata(metadata_filter)   # 内存过滤
   └─ state_filter 过滤                       # 内存过滤
   ▼
Json(Vec<ListedSandbox>)
```

### 实现细节（**最容易写错的部分**）

- **分页机制：没有真分页**。`nextToken` 是 schema 里的预留字段，CubeAPI **完全没读它**（`services/sandboxes.rs:86-93` 构造 `ListSandboxRequest` 时 `start_idx: Some(0)`、`filter: None`，不携带任何游标）。即便客户端传 `nextToken`，handler 也忽略。整个分页行为是「一次性拉取，limit 透传给 CubeMaster 的 `size`」。
- **`limit` 处理**：`size: Some(limit.max(1))`（`src/services/sandboxes.rs:90`）——最小 1，无上限；负数会被压成 1。`limit=0` 也会被压成 1（`i32::max(0, 1) == 1`）。这与 openapi schema 没声明 minimum 一致。
- **`metadata` 过滤语法**：`key1=val1&key2=val2` 形式，**多对用 `&` 分隔，必须全部命中**（AND），单值用 `=` 比较（`filter_by_metadata` `src/services/sandboxes.rs:755-775`）。注意：**这是 CubeAPI 内存过滤**——CubeMaster 拉回的是全量，CubeAPI 再筛。如果某 key 在 metadata 里不存在，认为不匹配。
- **`state` 过滤**：仅识别 `"running"` / `"paused"`，其它字符串（包括 `"pausing"`）会被 `parse_state_filter` 折成 `None`，**静默放弃过滤**（`src/services/sandboxes.rs:777-783`）。这一点在 openapi 没有文档化，调用方要小心。
- **字段映射**：`ListedSandbox` 由 `from_cubemaster_info` 组装（`src/services/sandboxes.rs:725-753`）：
  - `startedAt`：`started_at` → 回退 `create_at`（Unix 纳秒）→ 回退 `now()`；
  - `diskSizeMB`：**写死 `Some(0)`**（`src/services/sandboxes.rs:747`），不来自 CubeMaster；
  - `templateID`：先取显式字段，再回退到 annotation/label `cube.master.appsnapshot.template.id`（`extract_template_id`，`src/cubemaster/mod.rs:1206-1219`）；
  - `state`：由 `status` 字符串映射，**这条路径能暴露 `pausing`**（`sandbox_state_from_str`，`src/services/sandboxes.rs:797-803`）。
- **`volumeMounts`**：列表路径同样填 `None`（`src/services/sandboxes.rs:751`）。

### 关键代码引用

- handler：`src/handlers/sandboxes.rs:83-116`
- service：`src/services/sandboxes.rs:80-111`
- 过滤函数：`filter_by_metadata` `src/services/sandboxes.rs:755-775`、`parse_state_filter` `src/services/sandboxes.rs:777-783`
- 字段组装：`from_cubemaster_info` `src/services/sandboxes.rs:725-753`

---

## 6. `GET /v2/sandboxes/{sandboxID}/logs`

### 接口定义

- query 参数（`SandboxLogsV2Query`，`src/models/mod.rs:462-471`）：
  - `cursor`：`int64?`，**Unix 时间戳（秒级）** 而非 offset（见下）。
  - `limit`：`int32`，默认 1000（`default_log_limit`，`src/models/mod.rs:473-475`）。
  - `direction`：`string?`，**保留字段，未实现**——handler 与 service 都没读它。
- 返回 200 `SandboxLogsV2Response { logs: SandboxLogEntry[] }`（`openapi.yml:895-926`）；404 / 500。

### 数据流

```
HTTP GET /v2/sandboxes/{id}/logs?cursor=...&limit=...
   ▼  handler get_sandbox_logs_v2  src/handlers/sandboxes.rs:386-416
SandboxService::get_logs_v2(id, cursor, limit)  src/services/sandboxes.rs:397-432
   │  build_logs_request → SandboxLogsRequest { sandboxID, cursor, limit }
   │  POST /cube/sandbox/logs  src/cubemaster/mod.rs:180-193
   ▼
   ├─ 成功：ret.into_result → map SandboxLogLine → SandboxLogEntry
   ├─ is_endpoint_missing (HTTP 404 on path)：合成一条占位 Info 日志
   ├─ is_not_found (130404)：→ AppError::NotFound → 404
   └─ 其它：→ 500
```

### 实现细节

- **`cursor` 的真实格式**：是 **Unix 时间戳**（在 CubeMaster 侧的语义，由 `Option<i64>` 透传，`SandboxLogsRequest` `src/cubemaster/mod.rs:1348-1355`）。`SandboxLogsV2Query.cursor` 在 openapi 中标注为 `format: int64`，与代码一致。**不是字节 offset，也不是行号**。
- **`direction` 没实现**：`SandboxLogsV2Query` 里有该字段，但 `services::get_logs_v2` 与 `build_logs_request` 都没把它传给 CubeMaster（`src/services/sandboxes.rs:397-432`、`575-586`）。当前只能向前读取。
- **日志来源**：来自 **CubeMaster**（`POST /cube/sandbox/logs`），不是 envd。envd 是用户进程内的运行时守护，本接口暴露的是 CubeMaster 收集到的容器 stdout/stderr 转发。每行结构（`SandboxLogLine`，`src/cubemaster/mod.rs:1364-1370`）：`timestamp: DateTime<Utc>`、`message: String`、`level: String`。
- **`SandboxLogEntry` 结构**（`src/models/mod.rs:430-437`）：在 `SandboxLogLine` 基础上把 `level` 字符串映射为 `LogLevel` 枚举（debug / info / warn / error），未识别的统一归为 `Info`（`to_log_entry` `src/services/sandboxes.rs:813-826`）；`fields` **始终为空 map**（同上，`HashMap::new()`），是 v2 schema 上的预留扩展位。
- **endpoint-missing 兜底**：CubeMaster 的 `/cube/sandbox/logs` 在 `src/cubemaster/mod.rs:22-27` 的 module-level TODO 注释里**仍被列为「❌ New API required」**，但同一文件 `:1343` 的局部注释已改为「✅ Implemented」——**源码注释自相矛盾**，应以 `:1343` 为准（endpoint 已实现）。文档原作者采用了 module-level TODO 的旧视角，更稳妥的描述是：当前 endpoint 已实现，但 `is_endpoint_missing` 兜底路径仍保留，用于兼容老版本 CubeMaster（HTTP 404 on path 时，CubeAPI **不报错**，而是合成一条占位 Info 日志 `"(log streaming pending — CubeMaster endpoint not yet implemented)"`，`src/services/sandboxes.rs:417-425`）。
- **`SandboxLogsV2Response` 没有 nextCursor 字段**：v2 响应只有 `logs`，游标需要客户端根据最后一条日志的 `timestamp` 自行推算（`nextCursor = last_timestamp + 1`）。这是个值得在调用方文档里点出的「未文档化的契约」。

### 关键代码引用

- handler：`src/handlers/sandboxes.rs:386-416`
- service：`src/services/sandboxes.rs:397-432`
- CubeMaster logs RPC：`src/cubemaster/mod.rs:180-193`、`1345-1370`
- `SandboxLogEntry` 模型：`src/models/mod.rs:414-451`

---

## 共性设计

### 鉴权方式

统一中间件 `unified_auth`（`src/middleware/auth.rs:72-141`）：

- 若 `config.auth_callback_url` 未配置或为空：**全放行**（不校验任何凭据，默认部署形态）。
- 若已配置：
  1. 从 header 提取凭据，**优先级 `Authorization: Bearer <token>` > `X-API-Key: <key>`**（`extract_credential` `src/middleware/auth.rs:22-48`）。两者二选一，不重叠。
  2. 向 callback URL 发 POST，转发：
     - `Authorization: Bearer <token>` 或 `X-API-Key: <key>`（与客户端发的一致）；
     - `X-Request-Path: <原始路径>`；
     - `X-Request-Method: <HTTP method>`（关键，因为同一路径可能挂 GET/DELETE/POST，靠 method 区分读写）。
  3. callback 返回 200 → 放行；任何其它状态 → `AppError::Unauthorized` → HTTP 401；callback 不可达（网络错）→ `AppError::Internal` → HTTP 500（与 cluster / templates 类一致）。
- sandbox 路由同时挂 `rate_limit` + `unified_auth` 两层（`with_auth_and_rate_limit` `src/routes.rs:340-352`）；限流先于 auth 还是 auth 先于 rate_limit 见代码：tower layer 从外到内是 `.layer(rate_limit).layer(unified_auth)`，因此请求**先 auth 再 rate_limit**（auth 在最外层）。

### ApiError 模型与状态码映射

CubeAPI 内部错误类型 `AppError`（`src/error/mod.rs:13-36`），统一 `IntoResponse`（同文件 `38-51`）输出 `{ code: i32, message: string }`：

| AppError 变体 | HTTP status | code |
|---|---|---|
| `NotFound` | 404 | 404 |
| `Unauthorized` | 401 | 401 |
| `BadRequest` | 400 | 400 |
| `Conflict` | 409 | 409 |
| `Internal` | 500 | 500 |
| `TooManyRequests` | 429 | 429 |
| `NotImplemented` | 501 | 501 |

CubeMaster 业务码（`src/services/sandboxes.rs:25-29`）→ `AppError` 的映射分散在 service 各处：

- `0` / `200` → 成功；
- `130404` → `NotFound`（带 `"sandbox {id} not found"`，见 `sandbox_not_found_or_internal` `src/services/sandboxes.rs:656-662`）；
- `130409` → `Conflict`（在 `map_update_cubemaster_err` 与 `ensure_update_result` 里）；
- `130593`（master internal）→ 默认 `Internal`，仅 `kill_sandbox` 走二次探活路径可能升级为 `Conflict`；
- 其它非 0 → `Internal`。

### 路由版本（v2 前缀的含义）

`/v2/sandboxes` 与 `/v2/sandboxes/{id}/logs` 是「对外契约升级」路径：

- `list_sandboxes` v1 路径（`GET /sandboxes`，handler `src/handlers/sandboxes.rs:26-70`）写死 `limit=200`、不支持 state 过滤，**v2 把 limit / state / metadata 全部参数化**。
- 日志 v1 路径（`GET /sandboxes/{id}/logs`，handler `src/handlers/sandboxes.rs:339-369`）返回 `{ logs: [{timestamp,line}], logEntries: [...] }` 双结构（legacy E2B shape）；v2 统一为 `{ logs: [SandboxLogEntry] }` 单结构。
- v1 与 v2 **底层都打到同一个 CubeMaster RPC**（`POST /cube/sandbox/list` 或 `/cube/sandbox/logs`），差异只在 CubeAPI 的字段映射。

### 路由超时策略

- 所有 sandbox 生命周期路由走标准 30s 超时（`DEFAULT_ROUTE_TIMEOUT`，`src/routes.rs:26`，挂载在 `apply_http_layers` `354-363`）。
- snapshot create / rollback（`POST /sandboxes/{id}/snapshots`、`POST /sandboxes/{id}/rollback`）走 240s 长超时（`SNAPSHOT_LONG_ROUTE_TIMEOUT`，`src/routes.rs:38`），**与本类 6 个 API 无关**，但 mount 在同一路径前缀下，文档读者不要混淆。
- 超时触发返回 HTTP 408（`tower-http::timeout` 默认行为）。

### CubeMaster 调用：无重试 / 无客户端超时

- `CubeMasterClient` 是 `reqwest::Client` 的薄封装，**全文件无 retry / backoff / per-request timeout**（grep `src/cubemaster/mod.rs` 仅出现业务码 200、TTL 字段名）。一次 RPC 失败即映射为 `CubeMasterError::Http` / `Api` / `Deserialize`，由 service 层翻成 `AppError`。
- 唯一的「重试式」行为是 `kill_sandbox` 在 130593 时的二次 info 探活（见 §2），但那是「错误分类」，不是请求重试。
- 整个请求的超时上限**仅由 `tower-http` 路由层 30s/240s 控制**；CubeMaster 自身慢响应会先撞到这一层。这是设计上的有意取舍：避免长尾请求拖垮网关线程。

### 死代码 / 永不进入的分支

文档读者在读 `services/sandboxes.rs` 时容易被几处「写了但走不到」的代码误导：

- **`ensure_update_result` 的 `conflict_message` 参数**：`pause_sandbox` / `resume_sandbox` / `connect` 三个调用点都传入了 `"cannot be paused"` / `"is already running"` / 等具体文案（`src/services/sandboxes.rs:284`、`303`、`339`），看着像是 409 的回退文案。但 `parse_response`（`src/cubemaster/mod.rs:1620-1624`）在业务码非 0/200 时**直接抛 `CubeMasterError::Api`**，所以 `update_sandbox` 永远不会返回带 `130409` 的 `Ok(resp)`。`ensure_update_result` 实际只在 `ret_code == 0/200` 时跑（成功路径直接 `return Ok(())`），**404/409 分支永远走不到**。真正的 409 文案回退发生在 `map_update_cubemaster_err`（`src/services/sandboxes.rs:670-682`），统一回退到 `"sandbox {id} conflict"`（无 ret_msg 时）。`map_update_cubemaster_err` 上方的注释 665-666 明确点出了这个 quirk。
- **`ResumedSandbox.autoPause`**：handler 和 service 全程不读这个字段，源码已标 `#[allow(dead_code)]`（`src/models/mod.rs:322-323`），见 §4。

### 与 snapshot 子树的边界

`/sandboxes/{id}/snapshots`、`/sandboxes/{id}/rollback` 由 `snapshots` handler/service 处理（`src/routes.rs:162-174`），**不属于本类 API**。pause/resume 的底层机制是 container freezer（不依赖 snapshot），与快照子系统无耦合；snapshot service 文件中 grep `pause / resume / Pausing` 均无命中，可独立阅读。
