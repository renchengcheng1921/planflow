# AGENTS.md

本文件是给在本仓库中工作的 AI Agent（以及人类开发者）的操作手册。**开始任何编码前请完整阅读**。若本文件与实际代码冲突，以代码为准，并在 PR 中同步修正本文件。

---

## 1. 项目简介

PlanFlow 是「个人 TODO + 团队项目管理 + 计划可视化大屏」一体化工具。

核心产品原则（不可违背）：

1. **统一任务模型**：项目任务与每日 TODO 是同一个 `Task` 实体，禁止出现两套任务表/两套 DTO。
2. **今日清单克制**：默认限制今日任务数量，超出必须提示而非静默接受。
3. **通知默认少打扰**：重要事件强提醒，日常事件聚合摘要，禁止业务代码直接向用户发消息。
4. **大屏只读**：大屏所有接口只读、可脱敏、可 token 访问。
5. **所有变更留痕**：任何写操作都要产生 `Activity`。
6. **先闭环后智能**：AI/OKR 等属于 P2，不要在 MVP 阶段引入。
7. **敏感数据最小暴露**：凭据类数据（远程码、密码等）默认不可见、查看必审计、绝不进通知/日志/大屏。
8. **存储只用标准 S3 API**：对象存储为 RustFS（S3 兼容），代码不得依赖任何厂商私有扩展，保证可替换。

产品信息架构、四大核心需求、MVP 范围见 `docs/product/planflow.md`（即本仓库的产品方案原文）。

---

## 2. 快速开始

前置：Go ≥ 1.22、Node ≥ 20、pnpm、Docker（本地 PG/Redis/RustFS）。

```bash
make bootstrap                # 安装 go 工具链依赖 + pnpm install
cp .env.example .env          # 按需修改 DB_* / REDIS_* / STORAGE_* / CREDENTIAL_KEK
make up                       # docker compose 起 postgres / redis / rustfs / mailhog
make migrate                  # 执行 schema 同步（AutoMigrate + 手写迁移）+ 幂等 ensure bucket
make seed                     # 演示工作区与示例项目
make dev                      # 并行启动 api / worker / web
```

`make up` 会拉起 RustFS 容器（S3 API 默认 `http://localhost:9000`，控制台端口以官方镜像文档为准）。本地访问密钥通过 `.env` 注入容器与后端，**不要在代码或 compose 文件中硬编码默认密钥**。bucket 由 `make migrate` 幂等创建（HeadBucket → CreateBucket），不要求手动在控制台建桶。

### 常用命令

| 命令 | 作用 |
|---|---|
| `make dev` | 并行启动 `services/api`（:8080）、`services/worker`、`apps/web`（:3000） |
| `make build` | 编译 `cmd/api`、`cmd/worker`、`cmd/migrate` 并构建前端 |
| `make lint` | `golangci-lint run` + 前端 ESLint/Prettier |
| `make fmt` | `gofmt -w` / `goimports` + 前端格式化 |
| `make test` | `go test ./...` + 前端 Vitest |
| `make test:integration` | Go 集成测试（testcontainers 起 PG/Redis/RustFS） |
| `make test:e2e` | Playwright |
| `make migrate` | 执行 `cmd/migrate`（AutoMigrate + `migrations/*.sql` + ensure bucket） |
| `make seed` | 写入种子数据 |
| `make reset-db` | 重置本地数据库（**禁止在生产环境运行**） |

**提交前必须全绿**：`make lint && make test`。

---

## 3. 技术栈

### 后端

| 层 | 选型 |
|---|---|
| 语言 | Go（≥ 1.22），模块 `github.com/planflow/planflow` |
| HTTP | Gin |
| 数据库 | PostgreSQL |
| 数据访问 | **GORM v2**（`gorm.io/gorm` + `gorm.io/driver/postgres`） |
| 迁移 | **GORM AutoMigrate**（`cmd/migrate`）+ `migrations/*.sql` 处理 AutoMigrate 无法表达的场景 |
| 配置 | **Viper**（`spf13/viper`），启动时一次性加载并 `Unmarshal` 到强类型结构体 |
| 日志 | **Zap**（`go.uber.org/zap`），JSON 输出，请求级 Logger 注入 `context` |
| 校验 | `go-playground/validator`（tag）+ 关键业务规则在 domain 手写 |
| 异步/定时 | `hibiken/asynq`（Redis） |
| 实时 | SSE（`gin` 流式 handler）用于通知中心与大屏刷新 |
| 认证 | 会话/JWT + `auth` 包；大屏独立只读 token |
| 对象存储 | **RustFS**（S3 兼容），客户端用 `aws-sdk-go-v2`（`service/s3` + `feature/s3/manager`）（P1 引入） |
| 字段加密 | `crypto/aes` + GCM，信封加密（KEK/DEK），密钥版本化（P1 引入） |
| 测试 | `testing` + `testify` + `testcontainers-go` |

### 前端

| 层 | 选型 |
|---|---|
| 框架 | Next.js（App Router）+ React Server Components |
| 样式/组件 | Tailwind CSS + shadcn/ui |
| 数据请求 | Server Component 内 `fetch` 调 Go API；客户端用 TanStack Query |
| 局部状态 | Zustand |

新增依赖前先检查是否已有等价能力；新增运行时依赖需在 PR 描述中说明理由。**凭据加解密只允许在服务端完成，前端不得引入加密库自行解密。前端不得直接访问 RustFS，所有文件读写必须经后端预签名或代理。**

---

## 4. 目录结构

```
services/
  api/
    cmd/api/main.go          # HTTP 入口（配置加载、依赖装配、优雅退出）
    cmd/worker/main.go       # 异步任务入口（复用 internal）
    cmd/migrate/main.go      # 迁移入口：AutoMigrate + migrations/*.sql + ensure bucket
    internal/
      domain/                # 实体、值对象、不变式、领域错误（纯 Go，无 gin/gorm/s3 依赖）
      app/                   # 用例层：事务编排、权限、Activity 写入
      repo/                  # GORM 模型（models/）+ 仓储实现
      transport/http/        # gin handler、middleware、请求/响应 DTO、错误映射
      notify/                # 通知引擎（事件→规则→渠道→聚合）
      auth/                  # 认证、授权、策略（含字段级 action）
      storage/               # 对象存储抽象 + RustFS(S3) 实现（P1）
      secrets/               # 凭据加解密、KEK/DEK、密钥版本、reveal token（P1）
      platform/              # config(viper)、logger(zap)、clock、id、redis、mail/im 客户端
  migrations/                # 手写 SQL（AutoMigrate 无法表达的变更）
apps/
  web/
    app/(app)/today|inbox|projects|calendar|screen|reports|notifications|settings
    components/
docs/
  product/planflow.md
  decisions/                 # ADR
```

**分层规则（单向依赖）**：

```
transport/http  →  app  →  domain
                    ↓
                  repo / storage / secrets（接口在 app/domain 定义，实现在对应包）
```

- `domain` **不得** import `gin`、`gorm`、AWS SDK 或任何 `internal/repo|storage|secrets`。
- `app` 不直接接触 `gin.Context`，只接收 `context.Context` + 领域入参；**不直接持有 `*gorm.DB` 或 `*s3.Client`**，通过仓储/存储接口访问。
- `repo` 是唯一允许出现 GORM API 的层；`storage` 是唯一允许调用 S3 SDK 的层；`secrets` 是唯一允许加解密凭据的层。
- GORM 模型、S3 SDK 类型不得泄漏到 handler 的响应 DTO。
- handler 不写业务逻辑：只做「绑定 → 校验 → 调 app → 映射响应/错误」。

---

## 5. 领域模型与不变式（最重要）

核心实体：`Workspace`、`Project`、`Milestone`、`Task`、`TodayPlan`、`Notification`、`View`、`Activity`。

规划中（P1，见第 11 节）：`ProjectFile`、`ProjectCredential`、`CredentialAccessLog`。

### 必须遵守的不变式

1. **Task 唯一**：`Task` 通过 `project_id`（可空）+ `assignee_id` + `due_at` 等字段区分「项目任务」与「个人 TODO」。**新增任何"待办"实体前先证明无法用 Task 表达**。
2. **今日计划是引用，不是复制**：`today_plan_items` 只存 `task_id + 排序 + 时间块`，任务内容仍以 `tasks` 为准。删除计划项不等于删除任务。
3. **今日容量**：`TodayPlan` 默认上限 `MAX_TODAY_TASKS`（配置项，默认 7）。超出时 app 层返回 `ErrTodayFull`，handler 映射为 `409 TODAY_FULL`，前端弹确认，**不得静默写入**。
4. **状态与阻塞分离**：`Task.status` 为可配置状态流（默认 `TODO → IN_PROGRESS → REVIEW → DONE`），`blocked_reason`/`blocked_by_task_id` 是独立字段，不要为"阻塞"新增状态。
5. **依赖无环**：写入任务依赖前必须做环检测（`internal/domain/task/dependencies.go`），检测失败返回 `ErrDependencyCycle`。
6. **未完成滚动**：每日结算时未完成任务进入「待顺延」，需用户确认后加入下一日计划，**禁止静默丢弃或静默顺延**。
7. **通知必经引擎**：业务代码只能调用 `notify.Emit(ctx, event)`，由规则引擎决定渠道、级别、聚合、免打扰。**禁止在业务代码里直接调用邮件/IM/推送客户端**。
8. **变更留痕**：所有对 Project/Task/Milestone 的写操作必须在**同一事务**内写 `activities`（actor、action、before/after 摘要）。
9. **软删除**：使用 `gorm.DeletedAt`，删除进入回收站；查询默认自动过滤。永久删除只允许在回收站中且需二次确认，且必须显式 `Unscoped()`。
10. **时区**：数据库统一存 UTC（`timestamptz`）；展示与「今天」的边界按 `workspace.timezone` 计算，使用 `time.LoadLocation`，**禁止用 `time.Now()` 本地时区判断"今天"**。需要时间一律从 `platform.Clock` 取，便于测试。
11. **凭据永不明文**：任何凭据类字段（密码、远程码、密钥）落库前必须经 `secrets` 加密，**数据库中不得出现明文**；日志、通知、大屏、错误详情、Activity 的 before/after 中一律不得出现明文或密文。
12. **文件与凭据分离**：文档（PDR、变更文件）走 `ProjectFile` + RustFS；敏感字段走 `ProjectCredential` + 字段加密。**禁止把密码写进文件名、文件内容、任务描述或评论**。
13. **文件元数据是唯一事实源**：文件名、大小、checksum、版本、权限一律以 `project_files` 表为准；对象 key 是内部实现细节，**不得返回给前端，不得由前端拼装**。

### 权限

- 角色：`OWNER`、`ADMIN`、`MEMBER`、`VIEWER`（项目级继承工作区，可覆盖）。
- 所有读取/写入都必须经过 `auth.Authorize(ctx, actor, action, resource)`，**不允许在 handler 里手写散落判断**。
- action 采用 `资源.动作` 粒度（如 `task.update`、`file.download`、`credential.reveal`），**字段级敏感操作必须有独立 action**，不能复用 `project.read`。
- 大屏 token：只读、限定 workspace 范围、可脱敏、可撤销（`screen_tokens` 表，带过期时间）；**大屏 token 永远不具备 file/credential 相关 action**。

---

## 6. 数据库、GORM 与迁移

### GORM 使用约定

- **连接**：`platform/db` 统一构造 `*gorm.DB`，配置 `NamingStrategy{SingularTable: false}`、`NowFunc` 返回 UTC、`Logger` 适配到 Zap（按 `DB_LOG_LEVEL` 控制，生产默认只记慢查询与错误，**参数值不落日志**）。
- **模型位置**：GORM 模型集中定义在 `internal/repo/models/`，通过显式映射函数转换为 `domain` 实体。**禁止把带 `gorm` tag 的结构体直接当 API 响应**。
- **表名**：显式实现 `TableName()`，不依赖复数推导。
- **索引**：必须在结构体 tag 中显式声明（`index`、`uniqueIndex`），否则 AutoMigrate 不会创建。关键索引：`tasks(project_id, status)`、`tasks(assignee_id, due_at)`、`today_plans(workspace_id, plan_date)`、`notifications(user_id, read_at)`、`activities(resource_type, resource_id, created_at)`、`notification_dedup_key` 唯一索引。
- **事务**：写操作使用 `db.WithContext(ctx).Transaction(func(tx *gorm.DB) error { ... })`，Activity 写入与业务写入必须在同一事务。**禁止在事务内调用会自行开事务的仓储方法**（仓储需接受 `tx *gorm.DB` 参数以便组合）。
- **查询安全**：**禁止**用 `db.Raw` / `Where(fmt.Sprintf(...))` 拼接用户输入；动态排序/筛选字段必须走白名单映射。
- **Preload**：关联加载必须显式 `Preload`；列表接口禁止无脑全量 Preload 导致 N+1 放大。
- **条件复用**：可复用的查询条件用 `Session(&gorm.Session{})` 防止条件污染；`*gorm.DB` 不可跨 goroutine 共享。
- **批量写入**：批量插入使用 `CreateInBatches`（默认 500/批）。
- **幂等**：通知、重复任务、日报生成使用 `clause.OnConflict{DoNothing: true}` 或唯一键 + 冲突处理，**不得靠先查后写**。
- **错误**：仓储统一把 `gorm.ErrRecordNotFound` 转换为领域错误 `ErrNotFound`；禁止把 GORM 错误直接向上暴露。
- **禁用**：`AutoMigrate` 不得在 `api`/`worker` 启动流程中调用（仅限 `cmd/migrate` 与测试）。

### 迁移（AutoMigrate 为主）

- **正式同步入口**：`make migrate` → `cmd/migrate`，顺序为：① 执行 `migrations/*.sql`（按文件名排序）→ ② 对全部模型执行 `AutoMigrate` → ③ 幂等 ensure bucket。
- **生产**：`cmd/migrate` 作为独立部署步骤（Job / 一次性容器）执行，**不在应用进程内执行**；执行前需 review `AutoMigrate` 的 schema diff。
- **AutoMigrate 的能力边界（必须牢记）**：它能新增表/列/索引，但**不会**删除列、不会重命名列、不会改列类型、不会删索引、不做数据回填。以下变更**必须**额外写 `migrations/NNNN_xxx.sql`：
  - 删除/重命名列或表、改列类型、加约束（CHECK/FK）
  - 大表加索引（需 `CREATE INDEX CONCURRENTLY`）或加非空列（需分段：加可空列 → 回填 → 加 NOT NULL）
  - 任何需要数据回填的变更（**含凭据字段上线后的密钥轮换回填**）
- 手写 SQL 必须幂等（`IF NOT EXISTS` / `IF EXISTS`）或带版本记录表 `schema_migrations`（由 `cmd/migrate` 维护）。
- **不可回滚的迁移**必须在文件头写 `-- irreversible: 原因` 并在 PR 中说明。
- 数据库统一使用 `timestamptz`；Go 侧字段用 `time.Time`，写入前 `.UTC()`。

---

## 7. API 约定（Gin）

- 所有外部输入在 handler 绑定后校验；校验逻辑复杂时下沉到 `domain`，用 `fmt.Errorf("%w: ...", ErrValidation)` 包装。
- 统一错误响应：`{ "code": "...", "message": "...", "details": {} }`，`code` 为稳定枚举（`TODAY_FULL`、`DEPENDENCY_CYCLE`、`FORBIDDEN`、`VALIDATION_FAILED`、`NOT_FOUND`、`CONFLICT`、`REAUTH_REQUIRED`、`STORAGE_UNAVAILABLE`）。
- **错误映射集中在一处**：`transport/http/errors.go` 用 `errors.Is/As` 把领域错误映射为状态码，handler 只 `c.Error(err)`，**禁止每个 handler 自造状态码**。
- 中间件顺序固定：`request_id → recover → zap access log → auth → authorize → handler`。
- 列表接口必须分页（`limit` 默认 20、上限 100；游标或 `offset` 二选一，同一资源保持一致）。
- 大屏相关路由挂在只读 token 中间件下，不得注册任何写方法。
- 请求上下文与 DB 上下文一律用 `c.Request.Context()`；禁止用 `context.Background()` 逃逸请求生命周期（异步任务入口除外）。
- 响应 DTO 与 GORM 模型、领域实体、S3 SDK 类型分离，不直接序列化。
- **请求体日志白名单**：access log 中间件必须跳过或脱敏 `/projects/*/credentials*`、`/auth/*` 等路由的 body 与 query；**任何响应中出现凭据明文或预签名 URL 的接口，必须显式标注 `Cache-Control: no-store` 并禁止被缓存**。
- 文件上传/下载必须支持 `Content-Length` 与流式处理，**禁止把整文件读入内存后再转发**（大文件走 multipart 直传 + 后端校验，或后端流式代理）。

---

## 8. 配置（Viper）与日志（Zap）

### 配置

- 配置只在 `platform/config` 中加载**一次**，来源优先级：显式 `Set`（测试）> 环境变量（前缀 `PLANFLOW_`，`.` 替换为 `_`）> `config.yaml` > 默认值。
- 加载后必须 `Unmarshal` 到强类型 `Config` 结构体并执行 `Validate()`（必填项缺失直接启动失败）。
- 业务代码**禁止**调用全局 `viper.GetXxx`，必须通过注入的 `*config.Config` 读取。
- 密钥类配置（DB 密码、IM 凭据、token secret、**凭据加密 KEK**、**RustFS 访问密钥**）只允许来自环境变量或 KMS，不得写入 `config.yaml` 或仓库。
- 配置文件示例维护在 `config.example.yaml`；新增配置项需同时更新示例与本文件相关说明。

### 日志

- 统一从 `platform/logger` 获取；请求级 Logger 在中间件中携带 `request_id`、`user_id`、`workspace_id` 注入 `context`。
- **禁止使用 `fmt.Print*`、标准库 `log`**；`zap.L()` 全局单例仅允许在 `main` 装配阶段使用。
- 字段化输出：用 `zap.String("task_id", id)` 而非拼接字符串；键名统一 `snake_case`。
- 级别约定：`Error` 仅用于需要人工介入的失败；可重试的预期失败用 `Warn`；审计/变更留痕写 `Activity`/`CredentialAccessLog` 表而非日志。
- **禁止**记录：token、密码、远程码、KEK/DEK、密文、预签名 URL、对象 key、完整 Cookie、文件内容、超大 payload。
- panic 恢复中间件必须用 `zap` 记录堆栈并返回 `500`。

---

## 9. 前端约定

- 默认 Server Component；只有需要交互/浏览器 API 时才加 `'use client'`，并尽量下沉到叶子组件。
- 页面结构遵守产品定义：「我的今天」= 今日必做 / 时间块 / 收件箱待整理三块。
- 快速收集：全局快捷键（默认 `Ctrl/Cmd + K`）与悬浮按钮必须存在，收集默认进收件箱。
- 项目视图至少实现看板与列表；甘特图/时间线属 P1，未实现前不得放置假交互。
- 大屏（`/screen`）独立布局：只读、无写操作入口、支持全屏、轮播、刷新频率与脱敏开关；**大屏不提供文件/凭据卡片，也不允许通过配置开启**。
- 所有面向用户的文案走 i18n key（`messages/zh-CN.json`），不硬编码在组件中。
- 空状态、加载态、错误态是必需交付物，不允许只有 happy path。
- **敏感字段展示**：凭据字段默认掩码；明文仅在用户显式操作 + 二次认证后通过一次性接口返回，**不得写入前端持久缓存/URL/localStorage**，展示后自动隐藏。
- **文件访问**：前端只通过后端接口拿短期预签名 URL 或经后端代理下载；**禁止在前端构造对象存储 URL、暴露 bucket/对象 key、把 RustFS 控制台链接展示给用户**。预签名 URL 不得写入 `localStorage`/`sessionStorage`/日志/埋点。

---

## 10. 通知引擎约定

- 事件清单集中在 `internal/notify/events.go`，新增事件需同时更新规则默认值与文档。
- 处理链：`事件 → 规则匹配 → 级别 → 聚合窗口 → 免打扰检查 → 渠道投递 → 记录 Notification`。
- 默认策略：站内通知开启；邮件/IM 默认关闭，需用户显式订阅；免打扰时段内只保留 `URGENT`。
- 聚合：同一任务在聚合窗口内的多次变更合并为一条；升级规则只对 `URGENT` 生效。
- 投递由 worker 消费 asynq 任务完成，**handler 内不阻塞等待投递**；发送失败重试 + 死信告警，不得静默丢失。
- 日报/周报由 worker 定时生成，必须幂等。
- **通知内容不得包含**：凭据明文/密文、文件内容、预签名 URL、对象 key。文件/凭据更新类通知只允许提示「有更新，前往项目查看」，并可带项目/文件名（文件名需经脱敏策略）。

---

## 11. 项目文件与敏感凭据（P1，规划中，当前不实现）

> 本节描述后续要加入的能力（PDR、变更文件、现场设备 IP、向日葵远程码/密码等）。**当前阶段不实现任何相关代码或占位接口**，但领域抽象、权限模型、审计与配置设计必须不妨碍其落地。

### 11.1 两类数据必须分开建模

| 类型 | 实体 | 存储 | 典型内容 |
|---|---|---|---|
| 文档 | `ProjectFile` | **RustFS** + 元数据表 | PDR、变更文件、方案、截图、现场照片 |
| 凭据 | `ProjectCredential` | 字段级加密 + 元数据表 | 设备 IP、向日葵远程码、密码、账号 |

**禁止**把密码塞进文件（如 Excel/txt 上传）作为"绕过方案"；发现即视为数据泄露风险，需在 review 中驳回。

### 11.2 ProjectFile 约定（存储后端：RustFS）

- **客户端**：`aws-sdk-go-v2` 的 `s3` 包，**必须**配置 `BaseEndpoint = storage.endpoint` 与 `UsePathStyle = true`（RustFS 为自建 S3 兼容服务，不使用 virtual-hosted-style 域名）。
- **只用标准 S3 API**：`PutObject`、`GetObject`、`HeadObject`、`DeleteObject`、`ListObjectsV2`、`CopyObject`、`CreateMultipartUpload/UploadPart/CompleteMultipartUpload`、`HeadBucket`、`CreateBucket`。**禁止**使用 RustFS/MinIO 私有扩展、bucket notification 私有配置、对象锁等非通用能力；不依赖 bucket versioning（版本在应用层实现）。
- **不依赖 ACL**：访问控制全部由后端预签名 URL + `auth` 决定；**不使用对象级 ACL、不生成 public bucket**。bucket 默认私有。
- **寻址与 key 规范**：对象 key 形如 `ws/{workspace_id}/prj/{project_id}/{file_id}/v{version}`，**由服务端生成**，用户不可控；原始文件名只存元数据表，不放 key（避免注入与路径穿越）。
- **元数据**：`project_id`、`filename`（原始名）、`content_type`、`size`、`checksum`（SHA-256，**自行计算**，不依赖 ETag —— 分片上传的 ETag 不可靠）、`version`、`uploader_id`、`category`（PDR/变更/现场/其他）、`storage_key`（内部字段，不出 API）、`deleted_at`。
- **版本**：同名文档上传新版本不覆盖旧版本；保留最近 N 个版本（配置），历史版本可下载。旧版本清理由 worker 异步执行，失败可重试。
- **下载**：必须经 `auth.Authorize(action=file.download)`，返回短期预签名 URL（默认 5 分钟，`storage.presign_ttl` 可配），**禁止把永久 URL 或控制台地址暴露给前端**。下载完成后可选回调记审计；至少在上传、新版本、删除时写 `Activity`（含 IP、UA）。
- **上传限制**：单文件大小上限、类型白名单（配置）；上传采用分片直传或后端流式代理，**禁止整文件读入内存**；上传后异步做内容类型校验与病毒扫描（可选 ClamAV），扫描未通过的文件不可下载。
- **一致性**：写 `project_files` 与 `PutObject` 的先后顺序必须明确（推荐先落库 `pending` → 上传 → 置 `ready`；失败由补偿任务清理孤儿对象）。**禁止出现"对象存在但无元数据"或"元数据 ready 但对象缺失"长期不一致**，需有对账任务。
- **失败处理**：RustFS 不可用时返回 `503 STORAGE_UNAVAILABLE`，不得把 S3 SDK 错误原样透出；重试需有上限与退避。

### 11.3 ProjectCredential 约定

- **加密**：字段级 AES-256-GCM，信封加密。KEK 来自环境变量/KMS（`PLANFLOW_CREDENTIAL_KEK`），DEK 按项目或按记录生成并用 KEK 包裹；密文结构含 `key_version`，支持后续轮换。
- **字段结构**：`project_id`、`name`、`kind`（ip / remote_code / password / account / other）、`ciphertext`、`nonce`、`key_version`、`created_by`、`updated_by`、`expires_at`（可空）、`rotated_at`。
- **展示**：API 默认返回掩码（如 `10.0.*.*`、`******`）；明文通过独立接口 `POST /projects/:id/credentials/:cid/reveal` 返回，要求：
  1. `auth.Authorize(action=credential.reveal)`，默认仅 `OWNER`/`ADMIN`，其他角色需项目管理员显式授权；
  2. **二次认证**（密码或 TOTP，返回短期 `REAUTH_REQUIRED` 流程），re-auth token 单次有效、TTL ≤ 5 分钟；
  3. 响应头 `Cache-Control: no-store`，不写任何日志；
  4. 每次 reveal 写 `CredentialAccessLog`（who、when、ip、ua、credential_id、action=reveal）。
- **操作**：复制行为同样需要独立端点并审计（`action=copy`）；**前端禁止用剪贴板 API 直接复制掩码外的内容**，必须经服务端。
- **轮换**：支持手动轮换与到期提醒；轮换走同一加密通道，旧密文由迁移脚本或后台任务重加密。
- **导出**：默认禁止把凭据批量导出为文件；如未来支持，必须走审批流 + 加密导出 + 审计，**不得提供"一键导出全部密码"**。
- **删除**：软删除进入回收站，回收站中的凭据不可 reveal；永久删除需二次确认并留审计。

### 11.4 审计（不可省略）

- `CredentialAccessLog` 是 **append-only** 表：只允许 insert，不允许 update/delete（应用层禁止，必要时用 DB 权限约束）。
- 记录：`credential_id`、`actor_id`、`action`（reveal/copy/create/update/delete/rotate）、`ip`、`user_agent`、`created_at`、`result`（success/denied）。
- 审计日志本身不得包含凭据明文。
- 管理员可查看审计列表（只读），**审计日志不参与普通回收站机制**。

### 11.5 与大屏、通知、日志的边界

- **大屏**：不渲染、不聚合、不暴露文件/凭据的任何字段（含文件名、掩码、对象 key）。
- **通知**：只允许"项目有文件更新/凭据更新"的摘要提示，链接指向项目页，不携带内容。
- **日志**：`secrets` 与 `storage` 包内禁止打印明文、密文、预签名 URL、对象 key；相关 handler 在 access log 中过滤 body。

### 11.6 配置项（引入时新增）

| 配置 | 说明 |
|---|---|
| `storage.endpoint` | RustFS S3 API 地址，如 `http://rustfs:9000` |
| `storage.bucket` / `storage.region` | bucket 名与 region（RustFS 可用默认 region，但需显式配置） |
| `storage.access_key` / `storage.secret_key` | 仅来自环境变量，禁止入库/入仓 |
| `storage.use_path_style` | 固定 `true`（针对 RustFS/S3 兼容服务） |
| `storage.ensure_bucket` | 启动/迁移时是否幂等建桶，默认 `true` |
| `storage.max_file_size`、`storage.allowed_types` | 上传限制 |
| `storage.presign_ttl` | 预签名 URL 有效期，默认 5m |
| `storage.multipart_threshold` / `storage.part_size` | 分片上传阈值与分片大小 |
| `credential.kek` | KEK（只来自环境变量/KMS） |
| `credential.reauth_ttl` | 二次认证有效期，默认 5m |
| `credential.reveal_roles` | 允许 reveal 的默认角色 |

### 11.7 测试要求（引入时）

- `secrets`：加密/解密往返、篡改密文失败、密钥版本切换、错误密钥拒绝。
- `storage`：预签名 TTL、越权下载被拒、软删除后不可下载、对象 key 生成不可被用户输入影响、checksum 校验失败拒绝入库、分片上传往返。
- 权限：`MEMBER`/`VIEWER` reveal 被拒（403），未 re-auth 被拒（401 `REAUTH_REQUIRED`），大屏 token 访问文件/凭据接口被拒。
- 审计：每次 reveal/copy 产生一条 `CredentialAccessLog`，且日志与响应中无明文。
- 集成：用 testcontainers 启动 **RustFS** 容器验证上传/下载/分片/预签名链路；测试镜像 tag 必须锁定版本，**禁止 `latest`**。

---

## 12. 测试要求

| 范围 | 要求 |
|---|---|
| `internal/domain` | 表驱动单元测试：今日容量限制、未完成滚动、依赖环检测、通知规则匹配与聚合、权限判断、时区边界 |
| `internal/app` | 集成测试（testcontainers PG）：事务原子性（业务写入 + Activity 同时成功/回滚）、并发更新冲突、软删除过滤 |
| `internal/repo` | 查询/索引相关测试：分页边界、Preload 正确性、OnConflict 幂等 |
| `transport/http` | handler 测试：参数校验、错误映射、鉴权 401/403、大屏只读 405 |
| `secrets` / `storage`（P1） | 见 11.7 |
| E2E | Playwright 覆盖「收集 → 加入今日 → 完成 → 复盘」主闭环，以及大屏只读 |
| 回归 | 修复 bug 必须先补失败用例 |

- 测试优先使用真实 PG / RustFS（testcontainers），**不要用 mock 掩盖 SQL 层与对象存储层问题**；外部渠道（邮件/IM）用接口替身。
- 时间相关逻辑通过注入 `Clock` 控制，禁止在测试里 `time.Sleep` 等真实时间。
- 集成测试的 schema 由 `cmd/migrate` 逻辑统一创建，**禁止在测试里手写 `AutoMigrate` 子集**。
- 集成测试容器镜像 tag 必须锁定（PG、Redis、RustFS 皆是），避免 CI 因上游 `latest` 漂移。
- 新增业务规则但无测试的 PR 不予合并。

---

## 13. 提交与 PR 规范

- 提交信息遵循 Conventional Commits：`feat|fix|refactor|docs|test|chore(scope): summary`。
- 一个 PR 只做一件事；模型/迁移变更单独 PR 或至少在描述中显著标注。
- PR 描述必须包含：改动目的、影响范围、如何验证、是否涉及迁移/权限/通知、UI 截图（UI 改动）。
- **涉及 schema 变更**：PR 中必须贴出 `AutoMigrate` 会产生的影响（新增了哪些表/列/索引），以及是否存在需手写 SQL 的部分。
- **涉及凭据/文件（P1）**：PR 中必须说明加密方式、权限 action、审计点、日志过滤点、存储 key 规范、失败补偿策略，并附安全自查清单。
- 不要提交：`.env`、`config.yaml`（本地）、密钥、构建产物、本地数据库文件、测试用的 RustFS 数据目录。
- 不要顺手格式化无关文件；不要降低已有类型安全与错误处理标准。

### 完成定义（DoD）

- [ ] `make lint && make test` 通过
- [ ] 新逻辑有测试（domain 规则必测；事务/仓储改动有集成测试）
- [ ] 写操作在同一 GORM 事务内产生 Activity
- [ ] 外部输入已校验，错误已映射为稳定 `code`
- [ ] 权限检查覆盖新接口（`auth.Authorize`）
- [ ] 新增配置项已进入 `Config.Validate()` 与 `config.example.yaml`
- [ ] 日志无敏感信息、使用结构化字段
- [ ] 空/错/加载态齐全（UI）
- [ ] schema 变更已确认 AutoMigrate 可覆盖，否则已附带幂等 SQL 与说明
- [ ] （P1）凭据/文件相关：加密、re-auth、审计、日志/通知/大屏隔离均已验证
- [ ] （P1）存储相关：仅用标准 S3 API、path-style、预签名 TTL、对象 key 服务端生成、一致性/补偿已覆盖
- [ ] 文档与本文件在必要时已更新

---

## 14. 禁止事项（硬性）

1. 新建第二套任务/待办实体或 DTO。
2. 在 handler/组件中直接访问数据库或发送通知；或在 `repo` 之外的层使用 GORM API；或在 `storage` 之外的层使用 S3 SDK。
3. 绕过 `auth.Authorize` 或大屏只读约束。
4. 用 `time.Now()` 本地时区计算「今天」；或跳过 `Clock` 直接取时间。
5. 静默丢弃未完成任务、静默顺延、静默超限。
6. 大屏引入任何写接口或可编辑控件。
7. 在 MVP 阶段引入 AI、OKR、资源负载等 P2 能力；或提前实现文件/凭据功能的脚手架与占位接口。
8. 硬编码用户可见文案、时区、渠道凭据。
9. 在 `api`/`worker` 启动时执行 `AutoMigrate`；依赖 AutoMigrate 完成删列/改类型/数据回填等它做不到的变更。
10. 用 `db.Raw` / 字符串拼接构造含用户输入的 SQL；用 `Unscoped()` 做非回收站场景的删除/查询。
11. 全局调用 `viper.GetXxx`；用 `fmt.Print*`/标准库 `log` 打日志；记录敏感信息。
12. 忽略 `error`（`_ =` 需注释理由）、滥用 `panic`（仅 `main` 启动期可 panic）、在 handler 里逐个写状态码映射。
13. 使用 `context.Background()` 替代请求上下文（异步任务入口除外）。
14. 为通过检查而关闭 lint 规则（确有必要需 `//nolint:xxx // 理由` 并经 review）。
15. （P1）凭据明文落库/落日志/落通知；把密码放进文件或任务描述；把凭据或文件内容暴露给大屏；用普通 `project.read` 权限代替 `credential.reveal`；在 `secrets` 之外解密凭据；在前端保存凭据明文。
16. （P1）依赖 RustFS/MinIO 私有扩展或非标准 S3 能力；使用 virtual-hosted-style 寻址；开启对象级 ACL 或 public bucket；把对象 key、bucket 名、预签名 URL 暴露到前端、日志、通知或大屏；用前端传入的文件名拼对象 key。

---

## 15. 范围与优先级

- **MVP（当前目标）**：工作区/项目/任务/子任务；我的今天 + 收件箱 + 日历时间块；看板与列表视图；里程碑与健康度基础版；通知中心 + 到期/逾期/指派/@ + 每日摘要；大屏基础卡片（今日焦点、项目健康度、里程碑倒计时、逾期阻塞、动态流）；搜索/标签/评论/活动日志；模板/重复任务/回收站。
- **P1**：甘特图与依赖、自动化规则、多视图保存、工时统计、项目模板、API/Webhook、多工作区权限、导入导出。**项目文件（PDR/变更文件等）与敏感凭据（设备 IP/远程码/密码）**——按第 11 节实现，建议拆成两个独立 PR 序列：先 `ProjectFile`（含 RustFS 接入），后 `ProjectCredential`。
- **P2**：OKR、习惯打卡、资源负载、AI 辅助、风险登记册、审计日志（增强）、白板、客户只读分享页。

**未列入当前阶段的功能，不要提前实现脚手架或占位接口**，但领域抽象与权限模型不得阻断其加入。

---

## 16. Agent 工作流程

1. 读 `docs/product/planflow.md`、本文件、`docs/decisions/` 下相关 ADR。
2. 明确任务属于哪个阶段（MVP/P1/P2）；超出范围先提出，不要自行扩张。
3. 涉及模型变化：先在 `internal/repo/models/` 改模型（含索引 tag）→ 判断 AutoMigrate 是否足够，不足则写幂等 `migrations/NNNN_*.sql` → 更新 `cmd/migrate` 执行顺序 → 再改 `domain` → `app` → `repo` → `transport/http`，最后接前端。
4. 涉及业务规则：先在 `internal/domain` 写实现 + 测试，再在 `app` 编排（事务内写 Activity），最后接入 handler/UI。
5. 涉及配置/日志：先改 `platform/config`（含 `Validate` 与示例文件）或 `platform/logger`，再在业务代码中使用注入实例。
6. 涉及文件/凭据（P1）：先写 ADR（RustFS 接入与 key 规范、加密方案、权限 action、审计表、re-auth 流程、一致性/补偿），再按 `storage`/`secrets` → `domain` → `app` → handler → UI 顺序落地；每次提交前跑第 11.7 的安全测试清单。
7. 每完成一个可验证单元就运行 `go test ./internal/...`，不要攒到最后。
8. 无法确定的产品/技术决策：在 PR 中列出选项与推荐，标注 `TODO(decision)`，不要静默选择。
9. 提交前逐条对照第 13 节 DoD 与第 14 节禁止事项。

---

## 17. 术语表

| 术语 | 含义 |
|---|---|
| 收件箱 | 未整理事项的默认落点，快速收集入口 |
| 今日计划 / TodayPlan | 某一天的任务排序与时间块集合，引用 Task |
| 时间块 | 给任务分配的具体时间段 |
| 滚动未完成 | 当日未完成任务经用户确认后顺延 |
| 项目健康度 | 进度、逾期数、阻塞数、成员负载的综合指标 |
| 计划大屏 | 只读、可轮播、可脱敏的展示页面 |
| 聚合窗口 | 同一对象通知合并的时间范围 |
| 升级 | 逾期未处理时提高通知级别并通知上级角色 |
| PDR | 项目需求/设计类文档，作为 ProjectFile 的一种 category |
| 凭据 / Credential | 需加密存储的敏感字段（设备 IP、远程码、密码、账号） |
| KEK / DEK | 密钥加密密钥 / 数据加密密钥（信封加密） |
| reveal | 经二次认证后一次性查看凭据明文的行为，必审计 |
| re-auth | 敏感操作前的二次认证，结果以短期单次 token 表示 |
| RustFS | 本项目的 S3 兼容对象存储后端，仅使用标准 S3 API 接入 |
| 对象 key | 存储在 RustFS 中的内部路径，服务端生成，不对外暴露 |
| path-style | S3 寻址方式，RustFS 场景必须启用 |

---

如对本文件有疑问或需要变更约定，请新增 `docs/decisions/` 下的 ADR 并在 PR 中说明。