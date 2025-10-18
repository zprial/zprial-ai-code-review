# 部署与回滚作业手册

本手册指导 AI 代码审查平台在测试、预发、生产环境的部署流程，覆盖 CI/CD 阶段、凭证管理、灰度策略与回滚步骤。遵循本流程可确保上线安全、可追溯。

## 1. 环境与负责角色
- **测试环境**：日常功能验证；负责人 Dev 团队。
- **预发环境**：临上线验证，镜像生产配置；负责人 DevOps。
- **生产环境**：正式对外服务；负责人 DevOps + 运维值班。
- **值班体系**：参考 PRD RACI 表，严重告警时按“开发者 → 工程经理 → 运维”链路升级。

## 2. 前置检查
1. 确认主分支通过 `pnpm lint`, `pnpm test`, `pnpm test:integration`。
2. 数据库迁移已在测试/预发环境验证通过，并记录于变更单。
3. `.env.*` 配置及密钥经 KMS 注入且未过期。
4. 发布公告草稿已准备（含范围、风险、联系人）。

## 3. CI/CD 流水线概览
流水线由 GitLab CI 驱动，Stages 按顺序执行：

| 阶段 | 说明 | 关键命令 |
| --- | --- | --- |
| `lint-test` | 静态检查 + 单元/集成测试 | `pnpm lint && pnpm test && pnpm test:integration` |
| `build` | 打包服务、前端、Prisma 客户端 | `pnpm build` |
| `migrate` | 预演数据库迁移（测试/预发） | `pnpm prisma migrate deploy` |
| `deploy` | 通过 ArgoCD 部署到目标环境 | `kubectl/argocd app sync` |
| `post-verify` | 运行健康检查、冒烟测试 | 自定义脚本 `scripts/post-verify.sh` |

## 4. 部署步骤
### 4.1 测试环境
1. 合并至 `main` 触发流水线。
2. 自动部署到测试环境；观察 `post-verify` 结果。
3. QA/开发验证核心流程（MR 审查、评论生成、通知发送）。
4. 生效后更新变更记录。

### 4.2 预发环境
1. 在 GitLab 中手动触发 `deploy:staging` 并选择镜像版本。
2. 执行数据库迁移（若有），确认回滚脚本有效。
3. 运行 `scripts/staging-smoke.sh`，验证钉钉通知、报表、监控指标。
4. 与运维共签上线单，确认值班人员到位。

### 4.3 生产环境（灰度发布）
1. 通知相关团队上线时间，冻结非必要变更。
2. 触发 `deploy:prod`，指定与预发一致的镜像标签。
3. ArgoCD 按 10% → 50% → 100% 滚动更新，每步间隔 10 分钟，并验证：
   - 审查任务成功率
   - 超时/失败率无异常增长
   - 钉钉通知成功发送
4. 完成后发布上线公告，并在看板上登记版本号。

## 5. 回滚策略
### 5.1 条件
- 关键指标异常：任务成功率 < 95%，或超时率 > 5%。
- 审查结果错误导致阻断大量 MR。
- 外部 API 故障持续超过 30 分钟。

### 5.2 执行
1. 使用 ArgoCD 回滚到上一健康版本 `argocd app rollback ai-code-review <revision>`。
2. 运行 `pnpm prisma migrate resolve --applied <prev_migration>` 确认数据库回滚（若采用 down 脚本）。
3. 关闭告警并在事件记录中登记原因/解决方案。
4. 通知相关团队重新评估缺陷并规划修复版本。

### 5.3 回滚后验证
- 运行冒烟测试，确认核心功能恢复。
- 检查数据库一致性及审查队列状态。
- 在监控面板比对回滚前后指标趋势。

## 6. 监控与告警
- Prometheus 采集任务耗时、队列长度、失败/超时率、外部 API 调用成功率。
- Alertmanager 规则示例：
  - `review_failure_rate > 0.05 for 10m`
  - `external_api_errors > 20 within 5m`
- 告警推送：钉钉群 + PagerDuty；自动附带最新部署版本、负责人。

## 7. 发布后操作
1. 收集 24 小时内的运行数据，更新风险趋势报表。
2. 汇总问题并进入复盘：评估 AI 准确率、误报率与资源消耗。
3. 将改进项纳入下一迭代 backlog。
4. 更新 `docs/deployment-playbook.md` 与变更手册，确保经验沉淀。

## 8. 附录
- **脚本路径**：`scripts/post-verify.sh`, `scripts/staging-smoke.sh`, `scripts/rollback-check.sh`（待补充）。
- **联系人**：
  - 产品负责人（PO）
  - DevOps 负责人
  - 运维值班电话/钉钉账号
- **参考文档**：`docs/prd.md`, `docs/setup-local.md`, `docs/stories/3.3.observability-metrics-monitoring.md`

