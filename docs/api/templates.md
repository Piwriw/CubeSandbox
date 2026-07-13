# Templates 类 API 原理

> 一句话概述：这类 API 管理模板（沙箱的「黄金镜像」）的查询和兼容性运维。

> 同类文档：[health.md](./health.md) · [cluster.md](./cluster.md) · [sandboxes.md](./sandboxes.md)

## 接口清单

| 方法   | 路径                                                  | operationId                       | 简介                                              | 鉴权                                   |
|--------|-------------------------------------------------------|-----------------------------------|---------------------------------------------------|----------------------------------------|
| GET    | `/templates`                                          | `list_templates`                  | 模板列表（摘要）                                  | 见「共性设计 - 鉴权」                  |
| GET    | `/templates/{templateID}`                             | `get_template`                    | 模板详情（含 replicas、createRequest、网络字段） | 同上                                   |
| GET    | `/templates/compat`                                   | `template_compat`                 | 模板兼容性矩阵（节点维度 × 模板维度）             | 同上                                   |
| POST   | `/templates/compat/{templateID}/adopt-baseline`       | `adopt_template_compat_baseline`  | 把 UNKNOWN 状态的副本对齐到当前 baseline          | 同上                                   |

注：仓库根目录的 `openapi.yml` 在 `/templates` 这条路径上还挂了 POST/DELETE/PATCH、`/templates/{id}/builds/{buildID}` 等其它操作，但它们属于「模板生命周期（创建/重建/删除/构建追踪）」，不在本文档的「templates 类（查询 + 兼容性运维）」范畴内。

## 概念铺垫

- **模板（Template）**：CubeSandbox 中的「黄金镜像」。它由容器镜像构建而成，预先打包了 rootfs、agent、kernel 等运行所需组件，新建沙箱时直接以模板为底本复制启动，从而把冷启动降为毫秒级。模板 ID 由 CubeMaster 自动生成（`tpl-` 前缀，见 `CubeAPI/src/services/templates.rs:125-128`）。
- **模板与沙箱的关系**：沙箱是模板的一次实例化。同一份模板可以分发到多个节点上各持有一份副本（即下文 `replicas` 字段），沙箱创建时就近选取节点上的副本拉起。
- **为什么需要 compat 矩阵**：模板在分发到各节点后会「冻结」一组绑定的版本（`bound*`），但节点上 agent / guest image / kernel 是会持续滚动的（`current*`）。一旦节点版本与模板绑定的版本漂移（升级、回滚、缺副本），同一份模板在不同节点上拉起的沙箱就会出现行为不一致甚至起不来的情况。`compat` 接口正是把「模板绑定的版本」与「节点上当前版本」按节点维度做对比，输出一张可读的矩阵，给运维提供「哪些副本 stale、哪些 missing、哪些 unknown」的视图。`adopt-baseline` 则是对 UNKNOWN 副本的一次「对账」动作（见第 4 节）。

> 注：上述「副本」概念是基于 `replicas` 字段（`TemplateResponse.replicas: Vec<serde_json::Value>`，`cubemaster/mod.rs:1703`）和 compat 节点维度推断的，CubeAPI 这一层并不解读 `replicas` 的内部结构（注释明确写「Left as raw JSON to avoid coupling to CubeMaster-internal types」，见 `cubemaster/mod.rs:1700-1702`），具体副本语义在 CubeMaster 内部。

## 1. GET /templates

### 接口定义

- 路径：`GET /templates`
- 查询参数：`instance_type`（可选，string）。
  - `openapi.yml:268-275` 与 `CubeAPI/src/models/mod.rs:611-613` 都明确写：**"currently no server-side filter; reserved for future use"**。也就是说当前传入也不会按它过滤。`#[allow(dead_code)]` 标在 `CubeAPI/src/models/mod.rs:609` 的 `ListTemplatesQuery` struct 上（不是 handler 行），handler 里通过把参数命名为 `_params`（`handlers/templates.rs:38`）表达「未使用」——Rust 命名约定，不等同于 `#[allow(dead_code)]`。
- 响应：`200 OK` 返回 `TemplateSummary` 数组；`404` 表示后端 endpoint 不可用；`500` 为意外错误。

### 数据流

```
Client
  │  GET /templates
  ▼
axum Router → unified_auth middleware
  │  (可选 auth_callback_url 校验)
  ▼
list_templates handler (handlers/templates.rs:36-42)
  │  调 state.services.templates.list_templates()
  ▼
TemplateService::list_templates (services/templates.rs:37-58)
  │  调 cubemaster.list_templates(None, false)
  │     → GET {base}/cube/template
  ▼
CubeMasterClient::list_templates (cubemaster/mod.rs:322-337)
  │  reqwest GET + parse_response
  ▼
CubeMaster 内部（不在本仓库范围）
```

### 实现细节

- 调用 CubeMaster 时**不传 `template_id`、不传 `include_request`**（`services/templates.rs:40`），即走「列表模式」。
- CubeMaster 返回的 `TemplateListResponse.data`（类型 `Vec<TemplateSummaryItem>`，`cubemaster/mod.rs:1672-1678`）被逐项映射成 `TemplateSummary`（`services/templates.rs:44-57`）。
- **空串归一**：所有可空字段都经过 `non_empty()` 处理（`services/templates.rs:275-281`）——把空串/纯空白字符串转成 `None`，避免给前端写出 `"lastError": ""` 这种噪音。
- **不做缓存、不做合并、纯透传**：CubeAPI 只做字段裁剪和归一化。

### 关键代码引用

- Handler：`CubeAPI/src/handlers/templates.rs:36-42`
- Service：`CubeAPI/src/services/templates.rs:37-58`
- CubeMaster 客户端：`CubeAPI/src/cubemaster/mod.rs:322-337`
- 响应字段：`CubeAPI/src/models/mod.rs:618-635`（`TemplateSummary`），`cubemaster/mod.rs:1650-1667`（`TemplateSummaryItem`）

## 2. GET /templates/{templateID}

### 接口定义

- 路径：`GET /templates/{templateID}`
- 路径参数：`templateID`（string，必填），CubeMaster 端会校验路径段合法性（`cubemaster/mod.rs:551` 起的 `validate_path_segment` 只允许 `[A-Za-z0-9-]` 这类字符，拒绝 `_` `.` `:` 等）。
- 响应：`200` 返回 `TemplateDetail`；`404` 模板不存在；`500` 意外错误。

### 实现细节

- 调用 CubeMaster 走的是**同一个 `/cube/template` 端点**，只是带上 `template_id=<id>&include_request=true` 两个 query（`cubemaster/mod.rs:340-353`）。也就是说 CubeMaster 的 list/detail 共用一条路径，由是否有 `template_id` 区分。
- **404 判定**：CubeMaster 在「ID 不存在」时并不一定返回 404 业务码，CubeAPI 这边多做了一道兜底——如果返回的 `template_id` 与 `status` 同时为空，就主动返回 `NotFound`（`services/templates.rs:67-72`）。
- **从 `createRequest` 反解网络字段**：`networkType` 和 `allowInternetAccess` 不在 CubeMaster 顶层字段里，而是塞在 `createRequest` 这个 raw JSON 里。Service 层手动挖取（`services/templates.rs:74-92`）：
  - `networkType` ← `createRequest["network_type"]`（字符串）。
  - `allowInternetAccess` ← `createRequest["cube_network_config"]["allowInternetAccess"]`（布尔）。
  - 这是因为 `createRequest` 是当初创建模板时的原始 payload，CubeMaster 没有把它拆成顶层字段，CubeAPI 在视图层把它「拍平」出来方便前端展示。
- **status 字段取值**：CubeAPI 这一层没有枚举常量定义（透传 CubeMaster 字符串）。基于 `TemplateSummary.status: String`（`models/mod.rs:625`）以及创建/重建流程里的 `TemplateJob.status`（默认 `"accepted"`，`services/templates.rs:352`）可推断常见的取值至少包含：`accepted`、构建完成后的就绪态、以及错误态（与 `lastError` 同时出现）。**具体枚举依赖 CubeMaster 内部逻辑，本文档不展开**。
- **replicas 字段**：类型 `Vec<serde_json::Value>`（`models/mod.rs:649`，对应 `cubemaster/mod.rs:1703`），即「节点上各副本的元数据数组」。CubeAPI 刻意不为其定义强类型，注释明确写「Left as raw JSON to avoid coupling to CubeMaster-internal types」（`cubemaster/mod.rs:1700-1702`）。可以理解为「这份模板在哪些节点上各有一份可用的副本」。
- **createRequest 字段**：原始创建请求体（raw JSON，`models/mod.rs:650-651`）。前端可用于回显「当初是怎么建的」，也可作为再次重建时的参考输入。
- **jobID 字段**：最新一次 create/rebuild 的作业 ID（`models/mod.rs:662-663`，注释「Latest create/rebuild job id for the template」）。配合 `/templates/{id}/builds/{buildID}/status` 可以查构建进度。
- **lastError 字段**：最近一次失败的错误信息，构建成功时为 `None`（`non_empty` 归一后）。

### 关键代码引用

- Handler：`CubeAPI/src/handlers/templates.rs:58-64`
- Service：`CubeAPI/src/services/templates.rs:60-106`
- CubeMaster 客户端：`CubeAPI/src/cubemaster/mod.rs:340-353`
- 视图模型：`CubeAPI/src/models/mod.rs:639-664`（`TemplateDetail`）
- CubeMaster 响应模型：`CubeAPI/src/cubemaster/mod.rs:1684-1706`（`TemplateResponse`）

## 3. GET /templates/compat

### 接口定义

- 路径：`GET /templates/compat`
- 无入参（无 query、无 path）。
- 响应：`200` 返回 `TemplateCompatMatrixView`；`500` 意外错误。（注意：此路径**没有显式的 404**——因为整体矩阵的查询本身不会因为「单个模板不存在」而失败。）

### 数据流

```
Client
  │  GET /templates/compat
  ▼
template_compat handler (handlers/templates.rs:76-79)
  │  调 state.services.templates.compat_matrix()
  ▼
TemplateService::compat_matrix (services/templates.rs:236-243)
  │  调 cubemaster.get_template_compat()
  │     → GET {base}/cube/template/compat
  ▼
CubeMasterClient::get_template_compat (cubemaster/mod.rs:420-429)
  ▼
CubeMaster 内部汇总节点版本 + 模板副本状态，返回矩阵
```

### 实现细节

#### 顶层结构

返回体是一个 `{ summary, templates[] }` 结构（`models/mod.rs:827-831`）：

- `summary`：全集群汇总（见下）。
- `templates[]`：每个模板一行（`TemplateCompatRowView`），每行再嵌套 `nodes[]`（`TemplateNodeCompatView`）。

#### 节点维度：compatStatus 的判定依据

每个 `TemplateNodeCompatView`（`models/mod.rs:783-815`）携带成对的「绑定值」与「当前值」，三个组件各一对：

| 组件         | 绑定字段                  | 当前字段                  |
|--------------|---------------------------|---------------------------|
| guest image  | `boundGuestImageVersion`  | `currentGuestImageVersion`|
| agent        | `boundAgentVersion`       | `currentAgentVersion`     |
| kernel       | `boundKernelVersion`      | `currentKernelVersion`    |

外加 `compatStatus`（综合判定结果）与节点标识 `nodeID` / `nodeIP`。

> **重要**：`compatStatus` 的取值与判定规则由 CubeMaster 内部计算后下发，CubeAPI 这一侧**没有任何 enum 或常量定义**，只是 `pub compat_status: String`（`cubemaster/mod.rs:1749`、`models/mod.rs:790`）原样透传。下面这套取值规则是基于 `TemplateCompatSummary` 字段命名（`staleReplicas` / `missingReplicas` / `unknownReplicas`）和 `adopt-baseline` 端点文档（见第 4 节）**推断**出来的语义，**真实判定逻辑在 CubeMaster 内部**。

按 summary 字段命名反推，`compatStatus` 的可能取值至少包含以下四类：

| compatStatus（推断） | 含义（推断）                                                                                                                              |
|----------------------|-------------------------------------------------------------------------------------------------------------------------------------------|
| `OK`                 | 节点上既有该模板的副本，且 `bound*` 三件套与节点当前的 `current*` 一致。                                                                  |
| `STALE`              | 节点上有副本，但 `bound*` 与 `current*` 出现了漂移（节点侧版本滚动后比模板绑定的更新；或模板 baseline 变了但副本未对齐）。对应 summary 中的 `staleReplicas`。 |
| `MISSING`            | 按分布预期该节点上应该有副本，但实际没找到（副本丢失/未分发到位）。对应 summary 中的 `missingReplicas`。                                    |
| `UNKNOWN`            | 副本状态无法判定（比如刚加入集群、版本信息未上报、或 `bound*` 与 `current*` 中某一方缺失）。对应 summary 中的 `unknownReplicas`，也是 `adopt-baseline` 的修复目标。|

判定原则（**推断**）大致是：**对每个 (template, node) 二元组，比较模板绑定 baseline 与节点上报的当前版本三件套（agent / guest image / kernel），结合「该节点是否真的存在该模板副本」综合给出上述状态**。

#### 每行 `overall` 字段

`TemplateCompatRowView.overall: String`（`models/mod.rs:823`）是该模板在所有节点上的**整体状态汇总**（同样由 CubeMaster 计算，CubeAPI 透传）。**推断**：当一个模板所有节点都是 `OK` 时 `overall` 应为 `OK`；只要存在任意 `STALE`/`MISSING`/`UNKNOWN`，`overall` 即为对应的问题态。具体取值依赖 CubeMaster。

#### summary 各字段含义

`TemplateCompatSummaryView`（`models/mod.rs:769-781`）的五个字段（CubeMaster 端定义见 `cubemaster/mod.rs:1728-1740`）：

| 字段              | 含义（基于命名与上下文推断）                                                                                                              |
|-------------------|-------------------------------------------------------------------------------------------------------------------------------------------|
| `staleTemplates`  | 至少存在一个 `STALE` 副本的模板数（模板维度计数）。                                                                                        |
| `staleReplicas`   | 全集群 `STALE` 状态副本总数（节点×模板维度计数）。                                                                                         |
| `affectedNodes`   | 至少有一个副本处于非 `OK` 状态的节点数。                                                                                                  |
| `missingReplicas` | 全集群 `MISSING` 状态副本总数。                                                                                                           |
| `unknownReplicas` | 全集群 `UNKNOWN` 状态副本总数——**也就是 `adopt-baseline` 唯一直接影响的计数**。                                                            |

> 这五个计数全部由 CubeMaster 内部聚合好下发，CubeAPI 这一层（`services/templates.rs:299-307`）只是把 `stale_templates / stale_replicas / affected_nodes / missing_replicas / unknown_replicas` 五个字段做一一对应的字段重命名映射，**不参与任何计数逻辑**。

#### 容错

- 当 CubeMaster 返回 `data` 为 `None` 时，`compat_matrix()` 用 `unwrap_or_default()` 兜底为空矩阵（`services/templates.rs:242`），前端会拿到 `summary` 全 0、`templates` 为空数组。
- **不会像 cluster.version_matrix 那样**在 endpoint 缺失（404）时静默返回空（对比 `services/cluster.rs:55-61` 的特殊处理）；compat 这里**直接把错误透传出去**变成 500（`map_err(map_err)`，`services/templates.rs:241`）。

#### 与 cluster service 的关系

Compat 矩阵**不复用** `ClusterService::version_matrix`（`services/cluster.rs:55-61`）。后者是组件版本的「裸矩阵」（按 component × node 列出每个组件的所有版本及分布），前者是「模板视角」的兼容性判定结果。两者都依赖 CubeMaster 端掌握的节点版本数据，但接口和聚合维度完全不同，互不调用。

### 关键代码引用

- Handler：`CubeAPI/src/handlers/templates.rs:76-79`
- Service：`CubeAPI/src/services/templates.rs:236-243`（调用）、`299-333`（视图转换 `to_compat_matrix_view`）
- CubeMaster 客户端：`CubeAPI/src/cubemaster/mod.rs:420-429`
- CubeMaster 响应模型：`TemplateCompatMatrix` `cubemaster/mod.rs:1776-1782`；`TemplateCompatRow` `cubemaster/mod.rs:1764-1774`；`TemplateNodeCompat` `cubemaster/mod.rs:1742-1762`；`TemplateCompatSummary` `cubemaster/mod.rs:1728-1740`
- 视图模型：`models/mod.rs:769-835`

## 4. POST /templates/compat/{templateID}/adopt-baseline

### 接口定义

- 路径：`POST /templates/compat/{templateID}/adopt-baseline`
- 路径参数：`templateID`（string，必填）。
- 请求体：无。
- 响应：`200` 返回 `TemplateCompatAdoptResponseView { updated: i32 }`；`404` 模板不存在；`500` 意外错误。

### 数据流

```
Client
  │  POST /templates/compat/{templateID}/adopt-baseline
  ▼
adopt_template_compat_baseline handler (handlers/templates.rs:95-108)
  │  调 state.services.templates.adopt_compat_baseline(template_id)
  ▼
TemplateService::adopt_compat_baseline (services/templates.rs:245-256)
  │  构造 TemplateCompatAdoptRequest { action: "adopt_baseline", template_id }
  │  调 cubemaster.adopt_template_compat_baseline(&req)
  │     → POST {base}/cube/template/compat
  ▼
CubeMasterClient::adopt_template_compat_baseline (cubemaster/mod.rs:431-445)
  ▼
CubeMaster 把该模板下所有 UNKNOWN 副本对齐到当前 baseline，返回受影响数量
```

注意：兼容性查询 (`GET /cube/template/compat`) 与对账 (`POST /cube/template/compat`) 在 CubeMaster 那一侧共用同一条 URL，靠 HTTP method 和 body 中的 `action` 字段区分（`services/templates.rs:247` 把 `action` 写死成 `"adopt_baseline"`）。

### 实现细节

#### adopt 做了什么

- **目标**：把指定模板下所有 `compatStatus = UNKNOWN` 的副本，**对齐到「当前 baseline」**——也就是把节点上副本绑定的版本（`bound*`）刷新为节点当前实际运行的版本（`current*`），让副本重新被判定为 `OK`。
- **直接副作用**：仅仅更新 CubeMaster 内部关于「该模板在该节点上的副本绑定版本」这一**元数据/记账状态**，使下一次 `GET /templates/compat` 时这些副本的 `compatStatus` 从 `UNKNOWN` 转为 `OK`，`summary.unknownReplicas` 相应减少。
- **返回值 `updated`**：本次被对齐的 UNKNOWN 副本数量（`TemplateCompatAdoptResponse.updated: i32`，`cubemaster/mod.rs:1800-1808`）。如果该模板当前没有 UNKNOWN 副本，返回 0。

#### 同步还是异步？是否触发重建？

- **同步**：从 CubeAPI 视角看是一次普通的 `POST → 200 OK`，handler 等到 CubeMaster 返回后才响应（`handlers/templates.rs:99-107`），不像 create/rebuild 那样返回 `202 Accepted` 加 jobID（对比 `handlers/templates.rs:116-118`、`127-133`）。所以这是一个**同步完成的元数据更新**操作。
- **是否触发重建/重启沙箱**：**不触发**。`adopt_baseline` 改的是「副本绑定版本」这一兼容性记账字段，并不重建模板镜像，也不重启已经在跑的沙箱。也就是说：
  - 已经在运行的沙箱**不受影响**（它们用的还是各自的副本）。
  - 此后新建的沙箱会按对齐后的 baseline 拉起。
  - 不会产生新的 `TemplateJob`（对比 create/rebuild 都会返回 `TemplateBuildJob`，adopt 只返回一个 `updated` 计数）。

> 此处关于「不重建、不重启」的描述是基于接口形态（200 同步返回、无 jobID、字段语义）做出的**推断**。CubeMaster 内部是否在 adopt 时附带任何额外动作（比如通知节点 agent 刷新本地缓存），不在 CubeAPI 视野之内。

#### 何时使用

典型的运维流程：

1. `GET /templates/compat` → 看到 `summary.unknownReplicas > 0`。
2. 在矩阵中定位到 `compatStatus = UNKNOWN` 的模板 ID。
3. `POST /templates/compat/{templateID}/adopt-baseline` → 拿到 `updated` 数量。
4. 再次 `GET /templates/compat` 确认 `unknownReplicas` 归零、对应节点行回到 `OK`。

注意：adopt **只修 UNKNOWN**，对 `STALE` / `MISSING` 没有作用——这两种状态需要走重建（`POST /templates/{id}` redo）或副本重分发，不能靠 adopt 解决。

### 关键代码引用

- Handler：`CubeAPI/src/handlers/templates.rs:95-108`
- Service：`CubeAPI/src/services/templates.rs:245-256`
- CubeMaster 客户端：`CubeAPI/src/cubemaster/mod.rs:431-445`
- 请求/响应模型：`TemplateCompatAdoptRequest` `cubemaster/mod.rs:1794-1798`；`TemplateCompatAdoptResponse` `cubemaster/mod.rs:1800-1808`；视图 `TemplateCompatAdoptResponseView` `models/mod.rs:833-836`

## 共性设计

### 鉴权

所有 `/templates*` 路径都挂在统一的全局 `unified_auth` 中间件下（`middleware/auth.rs:72-141`），与其它类 API 共用同一套鉴权模型：

- **若 `config.auth_callback_url` 未配置**：所有请求直接放行（默认部署形态，不校验任何凭据；`auth.rs:78-81`）。
- **若已配置**：
  1. 从请求头取凭证，**Bearer 优先于 X-API-Key**（`auth.rs:22-48`）。
  2. 向 callback URL 发起 POST，转发 `Authorization`（或 `X-API-Key`）、`X-Request-Path`、`X-Request-Method` 三个头（`auth.rs:96-109`）。
  3. callback 返回 200 才放行；其它状态一律 `401 Unauthorized`（`auth.rs:121-140`）；callback 不可达（网络错）→ `AppError::Internal` → HTTP 500（与 cluster / sandboxes 类一致）。
- **关键设计点**：因为 `/templates/{id}` 这种路径上同时挂了 GET/POST/DELETE/PATCH，单看 path 无法区分读写，所以中间件**强制把 HTTP method 一起转发给 callback**（`auth.rs:67-71` 的 security note），让 callback 能做 path + method 粒度的细粒度授权。对本文档的四个接口而言：`GET /templates*` 是只读，`POST .../adopt-baseline` 是写操作（运维侧），callback 可以据此区别对待。

**限流策略**：templates 类路由走 `with_auth`（**不挂 rate_limit**），见 `routes.rs:210`。这与 sandbox 类（`with_auth_and_rate_limit`）不同——templates 是管理视图，调用频率由前端 Dashboard 控制，不需要网关侧限流。

### CubeMaster 客户端调用模式

- 所有 templates 类接口最终都走 `CubeMasterClient`（`cubemaster/mod.rs` 中的 `reqwest` 封装）发往 CubeMaster 的 `/cube/template*` 路径家族：
  - `GET /cube/template`（list & detail 共用，靠 `template_id` query 区分）
  - `GET /cube/template/compat`（矩阵查询）
  - `POST /cube/template/compat`（adopt，body 内 `action=adopt_baseline`）
- 客户端方法**不构造业务错误**，只把 HTTP/反序列化/业务码错误包成 `CubeMasterError`（`cubemaster/mod.rs:488-501`），由 service 层的 `map_err`（`services/templates.rs:259-269`）翻译成 `AppError`。
- 整个调用链是**纯转发**：CubeAPI 在 templates 这一类上几乎不做业务计算，唯一两处有「业务」的处理是：
  1. `get_template` 中从 `createRequest` JSON 里挖 `networkType` / `allowInternetAccess`（`services/templates.rs:74-92`）；
  2. `get_template` 的 404 兜底（`services/templates.rs:67-72`）。
- compat 矩阵的视图转换（`to_compat_matrix_view`，`services/templates.rs:299-333`）只做字段重命名 + `non_empty` 归一，**不做任何判定或聚合**——所有计数和状态判定都来自 CubeMaster。

### 错误处理

service 层统一的 `map_err`（`services/templates.rs:259-269`）：

| CubeMaster 错误                                  | 翻译成 CubeAPI 的 `AppError` | HTTP 状态 |
|--------------------------------------------------|-------------------------------|-----------|
| `InvalidPathParameter`（路径段非法字符）         | `BadRequest`                  | 400       |
| `is_not_found()`（业务码 130404）或 `is_endpoint_missing()`（HTTP 404，老版本 CubeMaster 没有这条路径） | `NotFound`                    | 404       |
| `is_conflict()`（业务码 130409，状态冲突）       | `Conflict`                    | 409       |
| 其它（HTTP 错误、反序列化失败、未知业务码）      | `Internal`                    | 500       |

- 对 templates 类四个接口而言：
  - `GET /templates`：可能 500（CubeMaster 故障）；handler 层声明 404 是为 endpoint 缺失预留。
  - `GET /templates/{id}`：可命中 404（模板不存在，service 层兜底）。
  - `GET /templates/compat`：基本只会 500（如前述，**不做 endpoint missing 静默兜底**）。
  - `POST .../adopt-baseline`：可命中 404（模板不存在），也可能因为模板当前不允许 adopt（如状态冲突）命中 409。
- 全程不抛 panic，错误都走 `AppResult<T>` → axum 的 `IntoResponse`，统一序列化为 `ApiError` JSON 体。
