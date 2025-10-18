# AI代码审查 Product Requirements Document (PRD)

## Goals and Background Context

### Goals
- 在 Merge Request 阶段识别并阻断高风险缺陷，向开发者提供可执行建议。
- 将人工审查遗漏的阻断级缺陷比例压缩至可接受范围（目标 <10%）。
- 将审查反馈时间控制在 5 分钟内，缩短开发者等待窗口。
- 为管理层提供按周/月聚合的风险趋势和责任分布，支撑质量治理决策。

### Background Context
内部开发团队需要在高频交付场景下维持代码质量与线上稳定性，现有人工审查流程耗时且缺乏一致性。过去的技术调研与头脑风暴明确了：AI 需要读取代码改动及必要上下文，输出证据化意见，按严重程度分级反馈，并提供趋势化可视分析。该产品将整合 GitLab/GitHub 工作流、钉钉通知以及风险报表，帮助团队在不增加审查成本的前提下预防线上事故。

### Change Log
| Date | Version | Description | Author |
| --- | --- | --- | --- |
| 2025-10-18 | 0.1 | 初始草稿 | PM John |

## Requirements

### Functional
1. **FR1:** 系统在 GitLab/GitHub Merge Request 创建或更新时自动触发代码审查，解析改动文件及最多两层关联依赖。
2. **FR2:** 对每个潜在问题生成分级评论（阻断/警告/提示/确认），包含代码引用、风险说明及修改建议，阻断级评论需自动 @ 提交者并阻止合并。
3. **FR3:** 支持钉钉群机器人，当阻断问题在设定时限内未解决时自动升级提醒指定人员。
4. **FR4:** 提供管理报表接口，按周/月统计风险项数量、严重度、提交者分布，并支持导出 CSV/JSON。
5. **FR5:** 允许配置和维护审查规则与 AI 模型参数，包括启用/禁用检测类别与阈值。

### Non Functional
1. **NFR1:** 95% 的审查请求需在 5 分钟内返回结果；超时需记录并上报。
2. **NFR2:** 阻断级问题识别的准确率需满足内部基线（初始目标：召回率 ≥80%，误报率 ≤15%），并具备后续迭代空间。
3. **NFR3:** 核心服务月度可用性 ≥99%，失败时需具备自动重试与人工告警机制。
4. **NFR4:** 审查过程中传输和存储的数据须遵循企业安全策略，不得泄露仓库外内容，并支持权限审计。

### Project Foundations & Environment
- **项目脚手架与初始化：** 第一次迭代需基于公司 Node.js/TypeScript 服务模板初始化仓库，包含标准目录结构、lint/format 配置、基础 README 以及示例健康检查路由。若模板不可用，则按《source-tree.md》手动创建 `apps/review-service`、`apps/web-console`、`packages/shared` 目录，并提交首个初始化 commit。
- **本地开发环境：** 统一使用 Node.js 20 LTS、pnpm 9 作为包管理器。提供 `docs/setup-local.md`（待建）说明包含：安装 Node 版本管理器、拉取仓库、执行 `pnpm install`、配置 `.env.local`、启动 `pnpm dev`。钉钉、GitLab/GitHub 凭证通过公司密钥管理服务 (KMS) 注入，文档需指向申请流程。
- **依赖安装与版本基线：** 首次提交需安装 `@octokit/rest`、`@gitbeaker/node`、钉钉机器人 SDK、`bullmq` 队列、`pg` 驱动、`zod` 校验库、内部 LLM SDK 等核心依赖，并在 `package.json` 内锁定次版本号。`pnpm install --frozen-lockfile` 作为 CI 依赖安装命令。
- **CI/CD 与部署流水线：** GitLab CI 负责三阶段：① `lint-test`（pnpm lint/test，含单元与集成测试）② `build`（打包审查服务与前端 bundles）③ `deploy`（使用公司 ArgoCD 管道部署至测试、预发、生产）。流水线执行前需注入密钥，从 KMS 拉取凭证；部署前执行数据库迁移、服务滚动发布与健康检查。
- **数据库与迁移策略：** 采用 PostgreSQL，使用 Prisma Migrate 管理 schema。每次迁移需具备 `down` 回滚脚本，生产环境执行前在预发验证。存储表包括 `review_tasks`、`issues`, `notifications`, `reports_cache` 等，数据脱敏遵循企业规范。若迁移失败，流水线自动回滚至上一版本并触发告警。
- **外部集成与失败恢复：** 为 GitLab/GitHub API、钉钉接口配置速率与错误监控：超出速率限制时指数退避重试三次；连续失败触发 Story 2.3 的告警，同时记录在 `external_failures` 表。提供临时开关以禁用外部通知，避免雪崩。
- **角色与值守：** 定义 RACI——PO 负责需求优先级，DevOps 维护部署流水线，审查工程师维护规则与模型，运维接受钉钉升级并跟踪阻断项。严重问题升级流程：AI → 开发者 → 阻断超时 → 工程经理 → 运维值班。文档需列出联系人与值守时间窗口。

## User Interface Design Goals

### Overall UX Vision
为开发者与管理层提供“无需跳出现有工具”的体验：开发者在 MR 评论流中看到结构化建议，管理层通过 Web 控制台查看风险趋势，同时保持信息简明、可追踪。

### Key Interaction Paradigms
- MR/PR 评论作为主反馈通道，使用分级徽标与模板化语句。
- 钉钉消息采用卡片式通知，突出未处理阻断问题。
- 管理控制台提供可筛选的表格与折线图，支持快速导出。

### Core Screens and Views
- 审查执行概览页：展示队列状态、最近审查结果。
- 风险趋势仪表板：按时间、仓库、严重度过滤展示趋势图。
- 配置管理页：管理审查规则、阈值与通知策略。

### Accessibility: WCAG AA

### Branding
遵循公司内部开发者平台的品牌色与组件库（若无则采用默认企业 UI 套件），确保与现有门户风格一致。

### Target Device and Platforms: Web Responsive

## Technical Assumptions

### Repository Structure: Monorepo
所有服务代码集中在单一仓库中，便于共享模型、审查规则与部署脚本。

### Service Architecture
采用模块化单体服务（可扩展为若干独立进程），提供：
- 审查触发与调度模块
- 代码解析与上下文拉取模块
- AI/规则引擎执行模块
- 报表与通知模块

### Testing Requirements
实施“Unit + Integration”策略：核心算法与适配器需单元测试，Git 平台与钉钉集成需集成测试，阻断逻辑需具备自动化回归集。

### Additional Technical Assumptions and Requests
- 语言首选 TypeScript/Node.js 服务侧，便于与 GitLab/GitHub API SDK 结合。
- AI 能力可调用内部 LLM 服务或可控第三方推理接口，需支持版本切换。
- 使用 PostgreSQL 或公司内统一数据平台存储审查结果与报表数据。
- 通过公司 CI/CD（如 GitLab CI）部署，支持灰度与回滚策略。
- 依赖 pnpm workspace 管理多包结构，统一 ESLint/Prettier 规则，提交前执行 `pnpm lint --fix` 与 `pnpm test`。
- 机密信息通过 Vault/KMS 注入，禁止写入仓库；提供 `secrets.example.yaml` 引导申请流程。
- 监控采用 Prometheus + Grafana，警报通过 Alertmanager ↔ 钉钉；日志接入 ELK，保留 90 天。
- 提前为未来扩展到 Bitbucket/Azure DevOps 预留抽象接口层，集成新增平台仅需实现 `RepositoryProvider` 适配器。
- 发布节奏遵循“测试 → 预发 → 生产”三段式，每次上线需附带版本公告模板与回滚步骤（详见待创建的 `docs/deployment-playbook.md`）。

## Documentation & Handoff
- **本地环境**：参考即将创建的 `docs/setup-local.md`，涵盖依赖安装、环境变量、启动与测试流程；Story 1.1 完成后需提交首版。
- **部署手册**：新增 `docs/deployment-playbook.md`，描述 CI/CD 阶段输入输出、灰度策略、回滚脚本及值守职责；与 DevOps 共建。
- **用户/运维指南**：在 Epic 3 完成后补充操作文档，覆盖仪表板使用、通知配置、常见问题与支持渠道。
- **变更沟通**：每次上线需发布变更公告模板（含影响范围、风险、联系人），由 PO 与工程经理联合审批。

## Epic List
1. **Epic 1: 基础接入与触发管线** — 建立与 GitLab/GitHub 的集成、审查触发与上下文采集能力。
2. **Epic 2: 智能审查与反馈分级** — 实现 AI 审查工作流、评论模板化输出、阻断机制与钉钉升级链路。
3. **Epic 3: 报表与运行质量治理** — 构建风险趋势报表、配置管理与可观测能力，支撑持续优化。

## Epic 1 基础接入与触发管线
建立最小可行的审查触发链路，确保在 MR 创建时能够解析改动、拉取上下文并排队执行。

### Story 1.1 建立仓库接入配置
As a 平台管理员,
I want 在配置界面或配置文件中注册 GitLab/GitHub 仓库与凭证,
so that 系统可以安全地监听目标仓库的 MR 事件。

#### Acceptance Criteria
1: 支持注册 GitLab 和 GitHub 仓库的访问凭证，并通过测试按钮验证有效性。
2: 配置保存后，系统能订阅 MR 创建/更新事件，失败时给出错误日志。
3: 权限不足或凭证过期时产生告警并要求重新配置。

### Story 1.2 解析改动与上下文拉取
As a 审查服务,
I want 在触发后抓取 MR 改动及一到两层关联文件,
so that AI 审查可以获得足够的上下文信息。

#### Acceptance Criteria
1: 能够列出改动文件，并对每个文件提取完整内容与差异。
2: 支持根据 import/调用关系额外拉取一层到两层关联文件，且可配置最大范围。
3: 解析结果以结构化数据形式存储，供审查引擎消费。

### Story 1.3 审查任务队列与超时处理
As a 平台管理员,
I want 管理审查任务队列并处理超时,
so that 审查能够按 SLA 可靠执行。

#### Acceptance Criteria
1: 建立任务排队机制，支持并发度配置与重试策略。
2: 超过 5 分钟的任务需自动标记为超时并产生告警。
3: 提供队列状态接口，展示正在执行、成功、失败和超时的任务数量。

## Epic 2 智能审查与反馈分级
实现 AI + 规则的审查能力，输出结构化反馈并连接升级通道。

### Story 2.1 构建审查规则与模型调用
As a 审查工程师,
I want 将静态规则与 AI 推理结合,
so that 能全面检测语法、逻辑、安全等问题。

#### Acceptance Criteria
1: 支持配置检测类别（语法、逻辑、安全、风格等）并组合执行。
2: AI 推理结果需包含疑似问题位置、理由与置信度。
3: 将规则与模型结果合并为统一结构，供后续分级与评论模板化使用。

### Story 2.2 生成分级评论与阻断机制
As a 开发者,
I want 在 MR 中看到分级评论并被阻断高风险合并,
so that 可以快速定位并修复问题。

#### Acceptance Criteria
1: 根据严重度映射输出阻断/警告/提示/确认四类评论，并引用具体行号。
2: 阻断级评论自动 @ 提交者并标记 MR 为需要变更后才能合并。
3: 评论文本包含风险说明、证据以及建议的修改方向。

### Story 2.3 钉钉升级通知
As a 工程经理,
I want 对长期未解决的阻断问题收到钉钉提醒,
so that 可以及时推进整改。

#### Acceptance Criteria
1: 支持配置通知群、@ 责任人名单及阈值时间（如 24 小时）。
2: 当阻断问题超过阈值未关闭时，自动发送钉钉卡片通知并附带问题详情。
3: 通知发送失败时产生日志并允许重试。

## Epic 3 报表与运行质量治理
提供持续监控与配置能力，帮助管理层掌握趋势、优化策略。

### Story 3.1 风险趋势报表
As a 管理层用户,
I want 查看周/月风险趋势与责任分布,
so that 可以评估质量改进效果。

#### Acceptance Criteria
1: 支持按时间、仓库、严重度筛选，并展示折线/柱状图。
2: 报表可导出 CSV/JSON，字段包含时间段、问题数量、提交者、严重度。
3: 数据刷新频率可配置（默认每日），并提供最后更新时间。

### Story 3.2 配置与参数管理界面
As a 审查工程师,
I want 调整规则、阈值和模型参数,
so that 可以根据反馈优化审查效果。

#### Acceptance Criteria
1: 提供基于角色权限的配置界面或 API，允许启用/禁用检测类别。
2: 参数调整需记录审计日志，包含操作者、时间、变更内容。
3: 配置变更后触发滚动生效并通知相关人员。

### Story 3.3 可观测性与指标监控
As a 运维工程师,
I want 监控审查服务关键指标,
so that 能提前发现异常并保持 SLA。

#### Acceptance Criteria
1: 暴露基础监控指标（任务耗时、队列长度、失败率、超时率等），可接入现有监控平台。
2: 当超时率或失败率超过阈值时自动告警。
3: 保存历史日志，支持按仓库、时间查询审查结果与告警记录。

## Checklist Results Report
尚未执行 pm-checklist；待文档定稿后运行并记录结果。

## Next Steps

### UX Expert Prompt
"请基于《AI代码审查 PRD》聚焦 MR 评论体验与风险报表控制台，输出前端交互蓝图，涵盖评论模板视觉要素、仪表板布局及钉钉通知样式。"

### Architect Prompt
"请依据《AI代码审查 PRD》设计系统架构，重点明确审查触发流水线、AI/规则引擎组合、数据存储与通知集成、监控与告警管道，以及扩展到其他代码托管平台的路线。"

### Post-MVP Outlook
- **平台扩展**：评估接入 Bitbucket/Azure DevOps 的可行性；通过 `RepositoryProvider` 适配器实现统一接口，目标在 MVP 后第一季度完成一项新增平台。
- **审查精度迭代**：收集阻断/警告误判数据，建立模型反馈回路，规划自动标注工具和模型再训练节奏。
- **自学习与团队定制**：探索项目级规则与模型权重调整，允许团队创建本地规则包或插件。
- **运维与成本优化**：引入任务优先级调度与推理缓存，持续评估 compute 成本；扩展告警至 PagerDuty 等渠道。
- **治理与合规**：接入安全审计、数据保留策略及审批链，确保跨团队推广时满足合规要求。
