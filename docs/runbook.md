# AI代码审查 运行手册 (Runbook)

本手册为运维与质量管理团队提供日常巡检、故障处理、发布验证及升级流程指引，结合《docs/prd.md》《docs/architecture.md》《docs/deployment-playbook.md》《docs/front-end-spec.md》使用。

## 1. 角色与联系方式
- **值班运维 (Primary Ops)：** _待填写_（钉钉、电话、邮箱）
- **工程经理 / Escalation Owner：** _待填写_
- **审查工程师 (Rules Owner)：** _待填写_
- **产品负责人 / 业务联系人：** _待填写_
- **Dev 当班支持：** 参考每周排班表（建议自动同步至钉钉群公告）

> 建议在每次上线前确认值班表，若无覆盖需重新指派。

## 2. 环境概览与访问
- **访问入口：** `https://ai-review.{env}.corp`（测试/预发/生产）
- **控制台登录：** 公司 SSO（需 `ai-review-{role}` 角色）
- **Kubernetes 命名空间：** `ai-review-{env}`
- **数据库实例：** PostgreSQL `ai_review_{env}`（只读连接用于排查）
- **监控面板：** Grafana 仪表板 `AI Review Overview`
- **日志查询：** Kibana 索引 `ai-review-*`
- **Alertmanager 路由：** `ai-review-ops` 钉钉机器人 + PagerDuty 服务

## 3. 日常巡检流程（建议每天/上线前执行）
1. **Dashboard 检查**
   - KPI 卡：成功率 ≥95%、平均耗时 <5min、阻断数趋势异常请记录。
   - “最新阻断”列表：确认是否存在 24 小时未处理的阻断。
2. **队列与任务**
   - 浏览 `/queues/status` 或 Grafana 队列面板，队列长度持续上升时提醒开发扩容。
   - 查看 `review_task_failed_total` 指标是否飙升。
3. **外部依赖**
   - Grafana 图表：GitLab/GitHub API 成功率、LLM 延迟、钉钉发送成功率。
   - 如出现 5 分钟内≥10 次 429/5xx，进入故障流程。
4. **通知升级**
   - 控制台「通知与升级」页面确认 webhook 正常、未过期。
   - Audit 日志检查最近 24 小时是否有设置变更。
5. **报表状态**
   - 风险趋势页面顶部的“最后刷新时间”应 ≤24 小时；超时请触发“立即刷新”。

## 4. 配置与通知操作指南
### 4.1 调整审查规则
1. 导航至「配置」→「检测类别」。
2. 修改阈值或启用/禁用规则，确保右侧摘要与期望一致。
3. 点击“保存变更”，在弹窗确认变更摘要。
4. 系统会自动记录审计日志并发送钉钉变更通知。
5. 若需回滚，使用顶部“回滚至上一版本”按钮。

### 4.2 模型参数管理
- 在「配置」→「模型参数」调整 LLM 温度、置信度阈值。
- 变更生效后观察 `llm_latency_seconds` 指标及阻断误判率。

### 4.3 钉钉升级策略
1. 前往「通知与升级」，核对 webhook、@人列表、阈值（小时）。
2. 通过“发送测试”验证连接，失败时查看 `notifications` 表或日志。
3. 若连续失败 ≥3 次，系统会自动告警；请在 15 分钟内确认并修复。

## 5. 发布与验证（概要）
详见《docs/deployment-playbook.md》。关键步骤：
1. **测试/预发**：流水线自动执行 `scripts/post-verify.sh`（冒烟校验）和迁移。
2. **生产灰度**：ArgoCD 滚动更新 10% → 50% → 100%，间隔 10 分钟。
3. **冒烟验证**（手动或脚本 `scripts/staging-smoke.sh`）：
   - 创建测试 MR 触发审查（确认评论生成）。
   - 模拟阻断超时，验证钉钉升级。
   - 检查 Grafana 指标、日志无异常。
4. **变更记录**：发布公告模板包含范围、风险、回滚方式、联系人。

## 6. 故障响应流程
### 6.1 Alertmanager 告警映射
| 告警 | 触发条件 | 处理动作 |
| ---- | -------- | -------- |
| `review_failure_rate` Warning | 失败率 >5% (10min) | 检查失败任务日志，联系开发分析根因 |
| `review_failure_rate` Critical | 失败率 >10% (10min) | 切换人工审查、视情况回滚 |
| `llm_timeout_rate` | LLM 超时 >20 次 (5min) | 启用规则-only 模式，联系模型团队 |
| `external_api_errors` | GitLab/GitHub 429/5xx >10 次 | 启用指数退避限流，通知平台支持 |
| `dingding_failure_total` | 通知失败持续 3 次 | 手动重试，确认 webhook/网络 |

### 6.2 阻断问题积压
1. 查看 Dashboard 阻断列表 → 联系对应提交者和工程经理。
2. 若超过阈值未处理，钉钉提醒应自动发送；如未发送，排查通知配置。
3. 需人工关闭阻断项或标记为“需要人工复核”，避免 KPI 偏差。

### 6.3 队列积压或服务不可用
1. `queue_length` 异常：扩容 K8s 副本（参考 `kubectl scale deployment/review-service --replicas=...`）。  
2. 检查 Redis 状态，必要时 Failover。
3. 若 Pod 大量 CrashLoop，看 `kubectl logs` 及 `review_service_error_rate`；必要时执行回滚脚本。

### 6.4 外部 API / LLM 故障
- 查看 `external_failures` 表与日志，判断是否平台限流。
- 启用“仅规则模式”（配置页开关），避免阻断逻辑完全失效。
- 告知利益相关方并记录事件。

## 7. 常见问题指南 (Troubleshooting)
| 场景 | 可能原因 | 处理步骤 |
| ---- | -------- | -------- |
| 评论缺少代码片段 | 上下文文件拉取失败 / 文件过大被截断 | 检查 `ingestion` 日志，调整上下文深度或手动补充建议 |
| 评论重复出现 | MR 更新未触发重新审查或 webhook 重试 | 手动触发 `POST /webhooks/...`，核对 GitLab/GitHub hook 状态 |
| 报表无数据 | 缓存过期未刷新 / 夜间批任务失败 | 检查 `reports_cache` 表，执行“立即刷新”或重跑批任务 |
| 钉钉通知不达 | Webhook 失效 / 签名错误 | 重新生成 webhook、更新配置、查看 `notifications` 表中的错误信息 |
| 指标不更新 | Prometheus scrape 失败 / 服务无指标 | 查看 ServiceMonitor 配置、确认 `/metrics` 正常响应 |

## 8. 脚本说明（补齐后更新）
- `scripts/post-verify.sh`：部署后自动冒烟，用于检查关键 API、队列、通知。
- `scripts/staging-smoke.sh`：预发环境的手动验证脚本。
- `scripts/rollback-check.sh`：回滚后确认服务恢复、迁移状态一致。

> 所有脚本执行结果需写入 CI 日志或记录在发布文档中。

## 9. 文档与参考
- 《docs/prd.md》— 产品需求、功能范围、Post-MVP 计划。
- 《docs/architecture.md》— 系统组件、数据模型、监控指标。
- 《docs/front-end-spec.md》— 控制台交互、组件与视觉标准。
- 《docs/deployment-playbook.md》— 发布流程、灰度策略、回滚方法。
- 《docs/setup-local.md》— 本地配置验证流程。

若需更新本手册，请在变更后于 Change Log 登记并通知运维/产品团队。
