# AI代码审查 Fullstack Architecture Document

## Introduction
This document outlines the complete fullstack architecture for“AI代码审查”，覆盖后端服务、前端控制台及其支撑的基础设施。方案基于 PRD（`docs/prd.md`）与配套文档（`docs/setup-local.md`, `docs/deployment-playbook.md`），为 AI 驱动的开发流程提供统一约束。我们采用自建 Node.js/TypeScript 单体服务 + Web 控制台的模式，确保在 GitLab/GitHub 工作流中嵌入自动化审查，并通过钉钉、报表与监控闭环治理。

### Starter Template or Existing Project
- 采用公司现成的 Node.js/TypeScript Monorepo 模板，提供 pnpm workspace、ESLint/Prettier 配置、基础 CI 脚本。
- 若模板不可用，按《source-tree.md》指南手动创建 `apps/review-service`、`apps/web-console`、`packages/shared` 结构；公共工具（API 客户端、领域模型）放在 `packages/shared`.

### Change Log
| Date       | Version | Description                         | Author        |
| ---------- | ------- | ----------------------------------- | ------------- |
| 2025-10-18 | 0.1     | 首版架构文档，基于 PRD v4           | Architect Winston |

## High Level Architecture

### Technical Summary
系统部署在公司 Kubernetes 集群，通过 GitLab CI + ArgoCD 实现三环境（测试/预发/生产）流水线。后端采用 NestJS（Node.js/TypeScript）实现审查触发、任务队列（BullMQ/Redis）、审查引擎编排、Webhook/API 集成与报表 API；前端控制台使用 Next.js 14 + React，为运维和管理层提供审查可视化、配置管理及报表导出。PostgreSQL 作为核心数据存储，Prisma ORM 负责 schema 管理。Prometheus/Grafana/Alertmanager 提供监控告警，ELK 负责日志。系统对接 GitLab/GitHub API、钉钉机器人、公司 LLM 推理服务与 KMS。架构满足 PRD 提出的 5 分钟 SLA、阻断级准确率及趋势可观测要求。

### Platform and Infrastructure Choice
- **Option A：公司自建 Kubernetes（推荐）**  
  - Pros：与现有 DevOps 流水线、KMS、监控体系无缝集成；允许 fine-grained 资源控制；满足内网安全要求。  
  - Cons：需维护集群与 ArgoCD 配置，运维成本更高。
- **Option B：AWS EKS + RDS + SNS/SQS**  
  - Pros：托管基础设施，易于横向扩展，集成 CloudWatch。  
  - Cons：需要额外合规审批；与现有 GitLab CI 集成变复杂；钉钉内网接入需跨云。

**Recommendation:** 采用公司自建 Kubernetes + ArgoCD + GitLab CI。

**Platform:** Internal Kubernetes Cluster  
**Key Services:** GitLab CI, ArgoCD, Kubernetes Deployments/Ingress, PostgreSQL PaaS, Redis, Vault/KMS, Prometheus/Grafana, ELK, DingTalk Robot, Internal LLM Gateway  
**Deployment Host and Regions:** 生产集群（华北多可用区），测试/预发在同一园区独立命名空间。

### Repository Structure
```
repo/
├─ apps/
│  ├─ review-service/        # NestJS 后端
│  └─ web-console/           # Next.js 控制台
├─ packages/
│  ├─ shared-domain/         # DTO、领域模型、类型定义
│  ├─ infra-clients/         # GitLab/GitHub/DingTalk/LLM 客户端
│  └─ ui-components/         # 前端复用组件（如图表封装）
├─ scripts/                  # 部署、回滚、健康检查脚本
├─ prisma/                   # Prisma schema 与迁移
├─ .github/.gitlab-ci.yml    # CI/CD
└─ docs/                     # PRD、架构、运维文档
```
使用 pnpm workspace 管理依赖，设置严格的路径别名与 lint 规则。

### Service Decomposition & Module Boundaries
| Service / Module          | Responsibility                                                                 | Notes |
| ------------------------- | ------------------------------------------------------------------------------ | ----- |
| `review-service`          | 审查任务接入、队列调度、上下文抽取、规则引擎、LLM 调用、反馈生成、报表 API     | NestJS 应用，划分为 `ingestion`, `analysis`, `feedback`, `reporting`, `config` 五个模块 |
| `web-console`             | 审查执行监控、风险趋势报表、规则/阈值配置、通知管理、运维面板                 | Next.js 14 App Router，使用 React Query + Zustand |
| `shared-domain`           | 通用 DTO、类型、校验 schema（zod）、业务枚举                                   | 被后端和前端复用 |
| `infra-clients`           | GitLab/GitHub/DingTalk/LLM API 客户端，封装认证、限流、重试                    | 使用 Axios + p-retry |
| PostgreSQL                | 存储任务、问题、通知、报表缓存、配置、审计日志                                 | Prisma 管理 |
| Redis (可选)              | BullMQ 队列存储、临时缓存                                                       | 高 SLA 阶段部署哨兵集群 |
| Prometheus/Grafana        | 采集并展示核心指标                                                              | Story 3.3 |
| Alertmanager + DingTalk   | 告警升级                                                                        | Story 2.3 & 3.3 |

## Frontend Architecture

### Responsibilities and Layout
- 使用 Next.js 14（App Router）实现 SSR/CSR 混合页面，保证管理台在后台登陆后能快速获取最新数据。
- 页面结构：  
  1. **Dashboard**：展示队列状态、审查成功率、超时率。  
  2. **Risk Trends**：折线/柱状图，按仓库、严重度、时间过滤。  
  3. **Configurations**：规则启用、阈值调整、模型参数设置。  
  4. **Notifications**：钉钉配置、升级规则管理。  
  5. **Audit & Logs**：展示审计记录、外部调用异常。
- 使用 Ant Design Pro 或公司内部组件库；图表采用 ECharts。
- 路由分层：`app/(dashboard)`, `app/reports`, `app/config`, `app/notifications`, `app/audit`.

### State Management & Data Access
- 全局数据使用 React Query（TanStack Query）处理异步和缓存；任务实时更新可使用 WebSocket 或 SSE（后端 Gateway 提供）。
- 局部表单状态使用 Zustand 或 React Hook Form。
- 授权信息通过 JWT/Session Cookie，前端在 `_middleware` 处校验。

### UI Patterns & Accessibility
- 标准化评论等级 badge（红色阻断、橙色警告、蓝色提示、绿色确认）。  
- 报表页面提供深色模式、键盘导航、WCAG AA 对比度。  
- 提供导出按钮和复制链接功能。

### Client-Side Error Handling
- 使用 Axios 拦截器处理 401/403，跳转登录。  
- 对 429/5xx 显示 Toast + 重试。  
- 捕获前端分析错误（如图表渲染失败）并上报 Sentry。

## Backend Architecture

### Service Composition
- NestJS 模块划分：  
  - `WebhookModule`: GitLab/GitHub 事件接收验证。  
  - `IngestionModule`: 差异解析、上下文扩展（调用 Git API、Git CLI）。  
  - `QueueModule`: BullMQ（Redis）任务入队、并发控制、超时检测。  
  - `AnalysisModule`: 静态规则 + LLM 推理；合并结果。  
  - `FeedbackModule`: 生成分级评论、调用平台 API、钉钉通知。  
  - `ReportingModule`: 报表查询、趋势聚合。  
  - `ConfigModule`: 阈值与规则配置、权限校验、审计。  
  - `MonitoringModule`: Prometheus 指标、健康检查。  

### API Endpoints
- RESTful API，统一 `/api/v1` 前缀。重点端点：  
  - `POST /webhooks/gitlab`, `POST /webhooks/github`: 事件接入。  
  - `GET /reports/overview`: 返回时间区间统计。  
  - `GET /reports/export`: 导出 CSV/JSON。  
  - `GET /configs/rules`, `PATCH /configs/rules`: 管理规则。  
  - `POST /configs/test-notification`: 钉钉通知测试。  
  - `GET /queues/status`: 队列状态。  
  - `GET /audits`: 审计记录分页。
- 后端同时提供内部 GraphQL 或 RPC 不必要，保持 REST + SSE 简洁。

### Async Processing & Scheduling
- BullMQ 队列 `review:tasks`，处理审查流程；指定并发（默认 5）。  
- Scheduler（Nest Schedule）处理阻断逾期扫描、报表缓存刷新、健康检查。  
- 每个任务生命周期：接收 → 上下文拉取 → 分析 → 评论输出 → 通知 → 结果写 DB。

### External Integrations
- **GitLab/GitHub**：通过官方 SDK（`@gitbeaker/node`, `@octokit/rest`）获取 diff、提交评论、设置阻断；带指数退避重试；速率超限时回退并记录 `external_failures`.  
- **LLM 服务**：调用内部 HTTP API，发送上下文 + 指令，设定超时 90s；失败后降级为规则结果+提示。  
- **DingTalk**：机器人 Webhook，发送 markdown/卡片消息。  
- **Vault/KMS**：在启动时拉取密钥、刷新凭证。  
- **Kubernetes**：使用 ConfigMap/Secret 注入配置，ServiceAccount 控制权限。

### Security & Compliance
- JWT + RBAC 控制前端访问，角色包括 `admin`, `reviewer`, `ops`.  
- Webhook 验证使用 HMAC secret（GitHub），GitLab 使用 Token/应用 ID。  
- 所有外部调用加签名/认证，敏感日志脱敏。  
- 数据库表包含审计字段（created_by, updated_by, last_access_at），满足合规审计。  
- 审查结果与日志保留策略：默认 180 天，可按需求归档/清理。

### Observability
- Prometheus 指标：`review_task_duration_seconds`, `review_task_failed_total`, `external_api_errors_total`, `queue_length`, `llm_latency_seconds`.  
- Grafana 仪表板：队列健康、任务成功率趋势、阻断/警告分布、外部 API 状态。  
- Alertmanager：  
  - 阈值：失败率 >5% (10 分钟), LLM 请求超时 >20 次（5 分钟），外部 API 429 >10 次。  
  - 通知：钉钉 + PagerDuty。  
- 日志：Winston + OpenTelemetry，输出 JSON 至 Logstash，索引按日期+环境。

### Scalability & Resilience
- 后端部署为 `Deployment` + HPA（CPU 60%/自定义指标），最低 2 个副本。  
- Redis 使用哨兵或 K8s StatefulSet，避免单点。  
- Postgres 采用 PaaS 主从架构，启用读写分离（默认写主库）。  
- 队列任务支持延迟与重试；如 LLM 限流，任务进入 `waiting` 并指数退避。  
- 启用 PodDisruptionBudget、优雅关闭（SIGTERM -> drain queue）。

### Developer Experience
- pnpm scripts：`pnpm lint`, `pnpm test`, `pnpm test:integration`, `pnpm dev`, `pnpm migrate`.  
- 使用 ESLint + Prettier + Husky pre-commit。  
- 提供 VS Code Workspace 设置、调试配置（`apps/review-service/.vscode/launch.json`）。  
- docker-compose 提供 `postgres`, `redis`, `llm-mock`, `webhook-mock` 组装本地平台。

### Technical Debt Controls
- 每次迭代评估阻断误判、超时率、LLM 费用；在报表中增加“模型表现摘要”。  
- 创建 `tech-debt.md` 记录待优化项（如 diff 解析性能、上下文深度策略）。  
- 定期回顾钉钉通知噪音，避免告警疲劳。

### Edge Cases & Known Risks
- 巨型 diff 或二进制文件：跳过并输出提示；限制单任务上下文大小（默认 1.5MB）。  
- LLM 响应异常：提供回退逻辑，仅输出规则结果并提示“需要人工复核”。  
- 外部平台 API 升级：通过 `infra-clients` 适配层隔离，定期跟进版本。  
- 队列积压：HPA + 手动扩容 Worker；提供 `scripts/queue-drain.sh`。  
- 数据敏感：严格控制日志，避免输出源代码；必要时仅记录 hash/片段。

### Architectural Patterns
- **Modular Monolith + Event-driven Sidecar**：后端核心为模块化单体，通过 BullMQ 处理异步任务，简化早期复杂度，同时保留未来拆分为微服务的能力。  
- **Hexagonal Architecture**：核心业务逻辑与基础设施适配器（Git 客户端、LLM、钉钉）分离，便于测试与替换。  
- **CQRS-lite**：写入（审查结果）与读取（报表）分离，使用 materialized view/缓存提升统计性能。  
- **Repository Provider Pattern**：抽象代码托管平台接口，保障后续扩展 Bitbucket/Azure DevOps 的可行性。

## Tech Stack

| Category             | Technology                 | Version | Purpose                                            | Rationale |
| -------------------- | -------------------------- | ------- | -------------------------------------------------- | ---------- |
| Frontend Language    | TypeScript                 | 5.x     | 类型安全、统一语言栈                              | 减少前后端上下文切换 |
| Frontend Framework   | Next.js                    | 14.x    | SSR/CSR 混合、App Router 提升 DX                   | 适合仪表板、易集成 API 路由 |
| UI Component Library | Ant Design + 内部组件库    | 5.x     | 快速构建表格/表单/图表                             | 与公司 UI 规范一致 |
| State Management     | React Query + Zustand      | 5.x / 4.x | 异步数据缓存 & 轻量状态管理                      | 满足实时仪表板需求 |
| Backend Language     | TypeScript                 | 5.x     | 统一语言、使用装饰器能力                           | 与前端共享模型 |
| Backend Framework    | NestJS                     | 10.x    | 模块化结构、内置 DI、Scheduler                     | 适合中大型服务 |
| API Style            | REST (JSON + SSE)          | -       | 简洁、兼容 Git 平台／控制台                        | 无需额外 gateway |
| Database             | PostgreSQL                 | 15.x    | 事务一致性、丰富查询                               | 复杂报表支持 |
| Cache/Queue          | Redis + BullMQ             | 7.x / 1.x | 任务队列、临时缓存                               | 保证 SLA |
| File Storage         | MinIO / 内部对象存储       | -       | 备份上下文快照（必要时）                           | 可选 |
| Authentication       | 公司 SSO + JWT             | -       | 统一身份体系                                       | 与现有权限系统兼容 |
| Frontend Testing     | Vitest + Testing Library   | 1.x     | 组件/页面单测                                       | 与 Vite/Next 集成好 |
| Backend Testing      | Jest + Supertest           | 29.x    | 单元/集成测试                                      | NestJS 默认生态 |
| E2E Testing          | Playwright                 | 1.48    | Web 控制台 & API 端到端场景                        | 跨浏览器可靠 |
| Build Tool           | pnpm                       | 9.x     | Dependency & build orchestrator                    | workspace 支持 |
| Bundler              | SWC + Turbopack (Next 内置) | -       | 前端编译                                           | 高性能 |
| IaC Tool             | Helm + Kustomize (可选)    | -       | Kubernetes 部署模板                                | 与 ArgoCD 集成 |
| CI/CD                | GitLab CI + ArgoCD         | -       | 自动化测试/部署                                    | 已在 PRD 约定 |
| Monitoring           | Prometheus + Grafana       | -       | 指标采集/可视化                                    | 公司标准 |
| Logging              | Elastic Stack (Filebeat → Logstash → ES → Kibana) | - | 集中日志 | 满足合规与检索 |
| CSS Framework        | Tailwind CSS（辅助）       | 3.x     | 优化样式迭代，配合 Ant Design 定制                 | 快速定制 |

## Data Models

### Core Tables (Prisma schema draft)
- **review_tasks**  
  | Field            | Type            | Notes                               |
  | ---------------- | --------------- | ----------------------------------- |
  | id (PK)          | UUID            | 任务 ID                             |
  | repo_provider    | Enum            | gitlab / github / future providers  |
  | repo_name        | String          | `<group>/<project>`                 |
  | mr_iid           | String          | MR/PR 编号                          |
  | status           | Enum            | pending / processing / success / failed / timeout |
  | severity_summary | Json            | 各级数量                            |
  | started_at       | Timestamp       |                                     |
  | finished_at      | Timestamp?      |                                     |
  | error_message    | Text?           | 失败原因                            |

- **issues**  
  | Field          | Type      | Notes |
  | -------------- | --------- | ----- |
  | id (PK)        | UUID      |                                      |
  | task_id (FK)   | UUID      | 关联 `review_tasks`                   |
  | severity       | Enum      | block / warn / info / confirm        |
  | file_path      | String    |                                      |
  | line           | Int?      |                                      |
  | message        | Text      |                                      |
  | suggestion     | Text?     |                                      |
  | source         | Enum      | rule / llm                           |

- **notifications**  
  | Field           | Type       | Notes                         |
  | --------------- | ---------- | ----------------------------- |
  | id (PK)         | UUID       |                               |
  | task_id (FK)    | UUID       |                               |
  | channel         | Enum       | dingding / email / pagerduty |
  | status          | Enum       | pending / sent / failed      |
  | attempt_count   | Int        |                               |
  | last_error      | Text?      |                               |
  | sent_at         | Timestamp? |                               |

- **reports_cache**  
  | Field           | Type       | Notes |
  | --------------- | ---------- | ----- |
  | id (PK)         | UUID       |                               |
  | dimension_hash  | String     | 时间段 + 仓库 + 过滤组合      |
  | payload         | Json       | 预计算结果                   |
  | refreshed_at    | Timestamp  |                               |
  | expires_at      | Timestamp  |                               |

- **configs_rules**（规则配置）
- **audit_logs**（记录所有配置变更、手动操作）
- **external_failures**（外部 API 调用异常列表）

Prisma schema 需加索引：`review_tasks(repo_provider, repo_name, started_at)`, `issues(task_id, severity)`, `notifications(channel, status, updated_at)` 等。

## API Design

### Webhook Ingestion
- `POST /webhooks/gitlab`  
  Headers：`X-Gitlab-Token` 验证；Body 包含 MR 事件。  
  响应：`202 Accepted` + 任务 ID。
- `POST /webhooks/github`  
  使用 HMAC SHA-256 验证 `X-Hub-Signature-256`。

### Review Workflow APIs
- `GET /queues/status` → 返回各状态数量、平均耗时。  
- `POST /queues/retry/:taskId` → 手动重试失败任务（需 admin 权限）。  
- `GET /reports/overview?from&to&severity&repository` → 统计数据。  
- `GET /reports/export?format=csv` → 导出报表。  
- `GET /configs/rules` / `PATCH /configs/rules` → 启用/禁用检测类别、阈值。  
- `POST /configs/test-notification` → 发送测试钉钉消息。  
- `GET /audits` → 审计日志分页。  
- `GET /metrics` → Prometheus 指标（受保护）。

### SSE / WebSocket
- `/streams/review-progress`：推送实时任务进度、LLM 状态；由 Dashboard 订阅。

## Integration Points
- **GitLab/GitHub**：读取 diff、提交评论、设置 MR 阻断；需支持 API 限流、分页 diff; pipeline 事件。  
- **DingTalk**：发送阻断逾期提醒，支持@责任人；失败重试和人工接管。  
- **LLM Gateway**：POST `/v1/review-judge`；若失败 fallback 到规则结果。  
- **KMS/Vault**：通过 sidecar 或 CLI 注入密钥；密钥轮换策略 90 天。  
- **Prometheus/Kibana**：指标/日志采集；提供标签 `env`, `service`, `repository`.

## Security & Compliance
- RBAC：`admin`（配置、策略调整）、`reviewer`（查看结果）、`ops`（监控与回滚）、`viewer`（只读报表）。  
- Secrets：托管于 Vault/KMS，容器通过 CSI Driver 挂载；在 GitLab CI 中使用 OIDC 获取临时凭证。  
- 输入验证：所有外部 payload 通过 Zod schema 校验；防止命令注入、路径遍历。  
- 审计：配置、手动重试、强制通过等操作记录到 `audit_logs` 并推送到 Logstash。  
- 数据保留：按照公司政策 180 天，提供清理 job（Prisma script）。

## Monitoring & Alerting
- 指标详见 Observability，小结：  
  - `review_service_error_rate` > 5% 触发 Warning，>10% Critical。  
  - `queue_length` 增长超过 15 分钟触发扩容建议。  
  - `dingding_failure_total` 升高时通知运维。  
- 仪表板模块：队列健康、任务耗时分位、外部依赖状况、LLM 使用、成本估算。  
- 日志：traceID 贯穿 webhook → 任务 → 评论 → 通知，便于根因定位。  
- SLO：95% 请求在 5 分钟内完成，失败率 <5%。通过 Prometheus Recording Rules 计算。

## Deployment & Operations
- GitLab CI pipeline：`lint-test` → `build` → `migrate` → `deploy` → `post-verify`.  
- ArgoCD 同步 Helm/Kustomize 模版；使用 Value 文件区分环境。  
- 灰度发布：K8s Deployment 分批滚动（10%→50%→100%），配合健康检查。  
- 回滚：`argocd app rollback`, Prisma `migrate resolve`, 并执行 `scripts/rollback-check.sh`.  
- 值守：运维接收 Alertmanager 告警，工程经理负责升级流程，开发者提供热修支持。

## Future Evolution
- 引入流式分析（LLM partial streaming）减少延迟。  
- 探索向量数据库缓存代码上下文。  
- 扩展支持 Bitbucket/Azure DevOps（实现新 `RepositoryProvider` 适配器）。  
- 结合 SonarQube/现有质量平台，实现多源规则合并。  
- 落地成本监控（LLM 调用费用、资源利用率）。

