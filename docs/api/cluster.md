# Cluster 类 API 原理

> 一句话概述：cluster 类 API 对外暴露集群的容量总览、节点清单（含健康/饱和度）、以及集群组件版本矩阵，供 Dashboard 的 Overview / Nodes 页面消费。数据全部来自 CubeMaster 的 `/internal/meta/*` 端点，CubeAPI 负责聚合、归一化和降级处理。

> 同类文档：[health.md](./health.md) · [sandboxes.md](./sandboxes.md) · [templates.md](./templates.md)

## 接口清单

| 方法 | 路径 | operationId | 简介 | 鉴权 |
| --- | --- | --- | --- | --- |
| GET | `/cluster/overview` | `cluster_overview` | 集群节点数、健康数、CPU/内存总量与可分配量、MVM 槽位上限 | 若配置了 `auth_callback_url` 则需 `Authorization: Bearer <token>` 或 `X-API-Key: <key>`，否则放行 |
| GET | `/cluster/versions` | `cluster_versions` | 集群组件版本矩阵（controlPlane + components + nodes 三维度） | 同上 |
| GET | `/nodes` | `list_nodes` | 全量节点列表（含容量、可分配、饱和度、conditions、本地模板、组件版本） | 同上 |
| GET | `/nodes/{nodeID}` | `get_node` | 单节点详情，字段同 `/nodes` 单项 | 同上 |

所有接口均为只读 `GET`，无查询参数（`get_node` 仅有路径参数 `nodeID`）。

---

## 1. GET /cluster/overview

### 接口定义

- 路径：`/cluster/overview`（同时挂在 `/cubeapi/v1/cluster/overview` 下，nest 在 `CubeAPI/src/routes.rs:49`，路由组装在 `routes.rs:253-267`）
- 查询参数：无
- 响应字段（`ClusterOverview`，`CubeAPI/src/models/mod.rs:841`）：

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `nodeCount` | int | 节点总数 |
| `healthyNodes` | int | 健康节点数（`node.healthy == true`） |
| `totalCpuMilli` | int64 | 全集群 CPU 总量（millicpu） |
| `allocatableCpuMilli` | int64 | 全集群当前可分配 CPU（millicpu） |
| `totalMemoryMB` | int64 | 全集群内存总量（MiB） |
| `allocatableMemoryMB` | int64 | 全集群当前可分配内存（MiB） |
| `maxMvmSlots` | int64 | 各节点 `maxMvmNum` 之和 |

### 数据流

```
HTTP GET /cluster/overview
   │
   ▼
[axum router]  build_cluster_routes → with_auth  (routes.rs:253-267, 328-338)
   │  若 auth_callback_url 配置则挂 unified_auth middleware
   ▼
[middleware::unified_auth]  (middleware/auth.rs:72)
   │  校验 Bearer / X-API-Key（callback 200 才放行）
   ▼
[handler::cluster::cluster_overview]  (handlers/cluster.rs:36)
   │  state.services.cluster.cluster_overview().await
   ▼
[services::cluster::ClusterService::cluster_overview]  (services/cluster.rs:27)
   │  1) cubemaster.list_nodes()        → GET /internal/meta/nodes
   │  2) fetch_used_resources()         → POST /cube/sandbox/list（聚合 running sandbox 占用）
   │  3) build_overview_with_used()     本地聚合
   ▼
JSON 200 ClusterOverview
```

### 实现细节

- **两路数据合并**：`cluster_overview` 先调 CubeMaster `/internal/meta/nodes` 拿节点清单（含 `capacity` / `allocatable` / `healthy` / `max_mvm_num`），再调 `/cube/sandbox/list` 拉所有 running sandbox 聚合出每台 host 的真实占用 `used_map`，最后用 `build_overview_with_used` 组装（`services/cluster.rs:27-31, 101-131`）。
- **`healthyNodes` 直接取 CubeMaster 给的 `node.healthy` 布尔字段累加**，CubeAPI 不做二次判定（`services/cluster.rs:110-113`）。健康判定逻辑位于 CubeMaster 侧。
- **`maxMvmSlots` 是各节点 `max_mvm_num` 的简单求和**（`services/cluster.rs:116`），并非取最小值或某种调度约束。
- **`allocatable` 的优先级**：若 `used_map` 命中该节点 `host_ip`，则用 `capacity - used`（下限 0）覆盖 CubeMaster 自报的 `allocatable`；否则回退到 CubeMaster 的 `allocatable` 字段（`services/cluster.rs:118-127`）。这样当 CubeMaster 的 allocatable 不准（例如尚未感知到刚启动的 sandbox）时，仍能给出接近真实的可调度余量。

### 关键代码引用

- `CubeAPI/src/handlers/cluster.rs:36-39` — handler 入口，直接转发到 service。
- `CubeAPI/src/services/cluster.rs:27-31` — 编排 list_nodes + used resources。
- `CubeAPI/src/services/cluster.rs:101-131` — `build_overview_with_used` 聚合循环。
- `CubeAPI/src/services/cluster.rs:66-90` — `fetch_used_resources` 只统计 `status == "running"` 的 sandbox，`cpu_count` 乘 1000 转 millicpu。
- `CubeAPI/src/cubemaster/mod.rs:450-459` — `list_nodes` 对应 `GET /internal/meta/nodes`。

### 错误码

| HTTP | 触发条件 | 来源 |
| --- | --- | --- |
| 401 | 鉴权 callback 非 200 / 缺失凭证 | `middleware/auth.rs:88-93, 129-140` |
| 404 | CubeMaster 返回 `ret_code == 130404`（节点未找到） | `services/cluster.rs:93-99` + `cubemaster/mod.rs:505-513` |
| 500 | HTTP 传输错、反序列化失败、其他 CubeMaster 错误 | `services/cluster.rs:97`（`AppError::Internal`） |

---

## 2. GET /cluster/versions

### 接口定义

- 路径：`/cluster/versions`
- 查询参数：无
- 响应：`VersionMatrixView`（`models/mod.rs:985`），分三个维度：

```
VersionMatrixView
├── controlPlane: ControlPlaneVersionView  { version, commit, buildTime }
├── components: ComponentMatrixRowView[]    // 按组件聚合
│     { component, declaredVersion, declaredVersions[], consistent, versions[] }
│       └── versions: ComponentVersionGroupView { version, nodes[] }
└── nodes: NodeVersionRowView[]             // 按节点展开
      { nodeID, healthy, components[] }
        └── components: NodeComponentEntryView { component, version, declared }
```

### 数据流

```
HTTP GET /cluster/versions
   ▼
[router + unified_auth]  (同上)
   ▼
[handler::cluster::cluster_versions]  (handlers/cluster.rs:89)
   ▼
[services::cluster::ClusterService::version_matrix]  (services/cluster.rs:55)
   │  cubemaster.get_version_matrix()  → GET /internal/meta/version-matrix
   │  Err(e) if e.is_endpoint_missing() → 返回空矩阵（降级）
   ▼
to_version_matrix_view  (services/cluster.rs:207)
   ▼
JSON 200 VersionMatrixView
```

### 实现细节

- **controlPlane 版本来源**：直接取 CubeMaster `/internal/meta/version-matrix` 返回的 `control_plane.{version,commit,build_time}`，代表集群控制面的"目标版本"（`services/cluster.rs:209-213`、`cubemaster/mod.rs:2098-2105`）。controlPlane 字段在 CubeAPI 层不做任何加工。
- **components 维度**：每个组件一行（`ComponentMatrixRowView`），包含 `declaredVersion`（单一声明版本）、`declaredVersions`（多版本声明数组）、`consistent`（是否所有节点跑在同一版本）、以及 `versions[]`（按版本分组，列出该版本跑在哪些 node 上，`ComponentVersionGroupView`）。这部分由 CubeMaster 聚合产出，CubeAPI 仅做 1:1 字段映射（`services/cluster.rs:214-231`）。
- **nodes 维度**：每个节点一行（`NodeVersionRowView`），列出该节点各组件的实际版本及 `declared` 布尔（表示该版本是否在 `declaredVersions` 内），同样来自 CubeMaster，CubeAPI 不做二次判定（`services/cluster.rs:232-247`）。
- **`consistent` / `declared` 的语义**：这两个字段是 CubeMaster 端计算好的，CubeAPI 只是透传。若需要复核一致性的判定规则，应到 CubeMaster 侧查。
- **降级策略（重要）**：当 CubeMaster 版本较老、尚未实现 `/internal/meta/version-matrix` 端点时，会返回 HTTP 404 或 `ret_code == 404`，此时 `is_endpoint_missing()` 为真，service 返回一个空的 `VersionMatrixView::default()` 而非报错，让前端平滑降级（`services/cluster.rs:55-61`、`cubemaster/mod.rs:530-537`）。

### 关键代码引用

- `CubeAPI/src/services/cluster.rs:55-61` — 404 端点缺失降级为空矩阵。
- `CubeAPI/src/services/cluster.rs:207-250` — 三维度 struct 1:1 映射。
- `CubeAPI/src/cubemaster/mod.rs:473-483` — `GET /internal/meta/version-matrix`。
- `CubeAPI/src/cubemaster/mod.rs:530-537` — `is_endpoint_missing` 判定（HTTP 404 或 ret_code 404）。

### 错误码

| HTTP | 触发条件 |
| --- | --- |
| 401 | 鉴权失败 |
| 500 | 端点存在但返回了 404 以外的错误（HTTP 错、ret_code 非 0 非 404、反序列化失败） |
| (无 404) | 端点缺失会被降级为 200 + 空矩阵，不会以 404 暴露给客户端 |

---

## 3. GET /nodes

### 接口定义

- 路径：`/nodes`
- 查询参数：无
- 响应：`NodeView[]`（`models/mod.rs:886`）。每个元素包含：

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `nodeID`, `hostIP`, `instanceType` | string | 节点身份 |
| `healthy` | bool | 健康（CubeMaster 给定） |
| `capacity` / `allocatable` | `NodeResourcesView` | CPU(milli) / 内存(MB) |
| `cpuSaturation`, `memorySaturation` | float | 0-100 占用百分比 |
| `maxMvmSlots`, `quotaCpu`, `quotaMemMB`, `createConcurrentNum` | int64 | 调度参数 |
| `heartbeatTime` | datetime? | 最近心跳 |
| `conditions` | `NodeConditionView[]` | kubelet 风格 condition |
| `localTemplates` | string[] | 节点本地缓存的模板 ID |
| `versions` | `ComponentVersionView[]` | 该节点各组件版本 |

### 数据流

```
HTTP GET /nodes
   ▼
[router + unified_auth]
   ▼
[handler::cluster::list_nodes]  (handlers/cluster.rs:52)
   ▼
[services::cluster::ClusterService::list_nodes]  (services/cluster.rs:33)
   │  1) cubemaster.list_nodes()       → /internal/meta/nodes
   │  2) fetch_used_resources()        → /cube/sandbox/list 聚合
   │  3) resp.data.into_iter().map(to_view_with_used)
   ▼
JSON 200 NodeView[]
```

### 实现细节

- **可分配量的覆盖逻辑**：与 overview 一致——若 `used_map` 命中 `host_ip`，则 `used = used_map`，`allocatable = (capacity - used).max(0)`；否则回退到 CubeMaster 自报的 `allocatable`（`services/cluster.rs:142-152`）。
- **饱和度算法**：`used = (capacity - allocatable).max(0)`，`saturation = (used / capacity * 100).clamp(0, 100)`，当 `capacity <= 0` 时直接返回 0（`services/cluster.rs:252-259`）。因此 `used > capacity` 的脏数据会被截断到 100%。
- **字段裁剪**：`conditions` / `localTemplates` / `versions` 三个数组都标注了 `skip_serializing_if = Vec::is_empty`，空数组不会出现在 JSON 中（`models/mod.rs:915-920`）。`localTemplates` 只保留 `template_id` 字符串列表，丢弃 path/media 等元信息（`services/cluster.rs:188-192`）。
- **无分页**：`/nodes` 一次返回全部节点，没有分页/过滤参数。

### 关键代码引用

- `CubeAPI/src/services/cluster.rs:33-41` — 编排 list + 聚合。
- `CubeAPI/src/services/cluster.rs:134-205` — `to_view_with_used`：字段映射 + 占用覆盖 + 饱和度计算。
- `CubeAPI/src/services/cluster.rs:252-259` — `saturation_pct` 工具函数。

### 错误码

| HTTP | 触发条件 |
| --- | --- |
| 401 | 鉴权失败 |
| 404 | CubeMaster `/internal/meta/nodes` 返回 `ret_code == 130404` |
| 500 | 其他后端错误 |

> 注：`fetch_used_resources` 失败时只打日志、返回空 map，不会让 `/nodes` 整体失败——此时各节点 `allocatable` 回退到 CubeMaster 自报值（`services/cluster.rs:66-77`）。

---

## 4. GET /nodes/{nodeID}

### 接口定义

- 路径：`/nodes/{nodeID}`，`nodeID` 为路径参数（string，required）
- 响应：单个 `NodeView`，字段同 `/nodes`。

### 数据流

```
HTTP GET /nodes/{nodeID}
   ▼
[router + unified_auth]
   ▼
[handler::cluster::get_node]  (handlers/cluster.rs:71)
   │  Path(node_id) 提取
   ▼
[services::cluster::ClusterService::get_node]  (services/cluster.rs:43)
   │  1) cubemaster.get_node(node_id)  → GET /internal/meta/nodes/{id}
   │  2) resp.data 为 None → AppError::NotFound
   │  3) fetch_used_resources() + to_view_with_used
   ▼
JSON 200 NodeView
```

### 实现细节

- **404 判定有两条路径**：
  1. CubeMaster 直接返回 `ret_code == 130404`（`is_not_found()`），service 的 `map_err` 把它转成 `AppError::NotFound`（`services/cluster.rs:93-99`、`cubemaster/mod.rs:505-513`）。
  2. CubeMaster 返回 200 但 `data` 字段为 `None`（`NodeResponse.data: Option<NodeSnapshot>`），service 主动构造 `AppError::NotFound("node {} not found")`（`services/cluster.rs:44-48`）。两者最终都映射到 HTTP 404 + `ApiError{code:404, message}`。
- **资源占用与 `/nodes` 完全一致**：同样调 `fetch_used_resources` 再走 `to_view_with_used`（`services/cluster.rs:48-49`）。
- **无节点存在性预检**：CubeAPI 不做本地缓存或 ID 校验，节点是否存在完全由 CubeMaster 决定。

### 关键代码引用

- `CubeAPI/src/handlers/cluster.rs:71-77` — handler。
- `CubeAPI/src/services/cluster.rs:43-50` — `data == None` 显式 404。
- `CubeAPI/src/cubemaster/mod.rs:461-471` — `GET /internal/meta/nodes/{id}`。
- `CubeAPI/src/cubemaster/mod.rs:2084-2092` — `NodeResponse.data: Option<NodeSnapshot>`。

### 错误码

| HTTP | 触发条件 |
| --- | --- |
| 401 | 鉴权失败 |
| 404 | CubeMaster `ret_code == 130404`，或 `data == None` |
| 500 | 其他后端错误 |

---

## 共性设计

### 鉴权方式

统一由 `middleware::unified_auth` 中间件处理（`middleware/auth.rs:72-141`），逻辑如下：

1. 若 `config.auth_callback_url` 未配置或为空 → 直接放行（默认部署形态，不校验任何凭据）。
2. 否则从请求头提取凭证，优先级 `Authorization: Bearer <token>` > `X-API-Key: <key>`（`extract_credential`，`middleware/auth.rs:22-48`）。
3. 向 callback URL 发 POST，带上原凭证头、`X-Request-Path`、`X-Request-Method`；callback 返回 200 才放行，否则 401（`middleware/auth.rs:98-140`）。
4. callback 不可达（网络错）→ 500 `Internal`。

是否挂鉴权层由 `with_auth(routes, state, auth_configured)` 控制（`routes.rs:328-338`）：`auth_configured = config.auth_callback_url.is_some_and(|u| !u.is_empty())`（`routes.rs:41-45`）。

**限流策略**：cluster 类路由走 `with_auth`（**不挂 rate_limit**），见 `routes.rs:266`。这与 sandbox 类（`with_auth_and_rate_limit`，`routes.rs:340-352`）不同——cluster 是只读管理视图，调用频率由前端 Dashboard 控制，不需要网关侧限流。

### CubeMaster 客户端注入

- `CubeMasterClient` 是 `reqwest::Client` + `base_url` 的轻封装，`Clone` 为 O(1)（内部 `Arc` 连接池，`cubemaster/mod.rs:42-55`）。
- 在 `AppState::new` 中由 `config.cubemaster_url` 构造（`state.rs:52`），随后传入 `AppServices::new`（`services/mod.rs:92-107`），每个 service 各持一份 clone。
- `AppServices.cluster: ClusterService` 通过 `state.services.cluster` 暴露给 handler（`services/mod.rs:85, 94`）。
- 路由挂载见 `routes.rs:253-267`（`build_cluster_routes`），统一 nest 在 `/cubeapi/v1` 下（`.nest` 在 `routes.rs:49`，子路由组装在 `routes.rs:83-91` 的 `build_cubeapi_router`）。

### 错误模型

- service 层用 `AppError`（`error/mod.rs:13-36`），handler 统一返回 `AppResult<impl IntoResponse>`。
- `CubeMasterError → AppError` 的映射在 `services/cluster.rs:93-99`：`is_not_found() || is_endpoint_missing()` → `NotFound`，其他 → `Internal(anyhow)`。
  - 注意：`map_err` 只在 list/get 路径生效；`version_matrix` 自行处理 `is_endpoint_missing` 为降级而非错误。
- `AppError::IntoResponse` 把每个变体映射到 `(HTTP status, int code, message)`，并以 `ApiError{code, message}` 作为 JSON body（`error/mod.rs:38-51`）。
- CubeMaster 的 `parse_response` 会把 `ret.ret_code != 0/200` 的逻辑错提为 `CubeMasterError::Api{ret_code, ret_msg}`（`cubemaster/mod.rs:1598-1643`），`ret_code == 130404` 即"未找到"，`ret_code == 404` 或 HTTP 404 即"端点不存在"。

### 缓存 / 性能取舍

- **无缓存**：cluster 类 4 个接口每次请求都会实时打 CubeMaster，CubeAPI 层不做 TTL 缓存。其中 overview / list_nodes / get_node 三个接口还会额外打一次 `/cube/sandbox/list`（固定 `size: 500`、`start_idx: 1`，`services/cluster.rs:67-74`）来计算真实资源占用。
- **N+1 风险**：`fetch_used_resources` 一次性拉最多 500 条 sandbox（CubeMaster 不按状态过滤，全量返回前 500 条），CubeAPI 在客户端 `filter status == "running"` 后再聚合。当集群 sandbox 总数 > 500 时当前实现只看到前 500 条里的 running 副本，存在统计偏差——这是一个已知取舍，换取单次请求的简单性。
- **降级优先**：`fetch_used_resources` 任何错误都返回空 map，主流程不中断；`version_matrix` 端点缺失则返回空矩阵。整体设计偏好"返回部分正确数据"而非"硬失败"。
- **连接复用**：所有 CubeMaster 调用共享同一个 `reqwest::Client`（连接池 `pool_max_idle_per_host(100)`，`state.rs:46-50`），避免握手开销。
- **无重试 / 无客户端超时**：`CubeMasterClient` 是 `reqwest::Client` 的薄封装，**不带 backoff / retry / per-request timeout**（与 sandboxes 类一致，详见 `sandboxes.md` 共性章节）。一次 RPC 失败即翻成 `AppError`。
- **路由超时**：cluster 类接口走标准 30s 路由超时（`DEFAULT_ROUTE_TIMEOUT`，`routes.rs:26`，挂载在 `apply_http_layers` `:354-363`）。CubeMaster 慢响应会先撞到这一层，返回 HTTP 408。
