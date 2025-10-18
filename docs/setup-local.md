# 本地开发环境搭建指南

## 1. 前置条件
- macOS 13+/Ubuntu 22.04+（其他 Linux 需自测）
- Node.js 20.x LTS（推荐使用 nvm）
- pnpm 9.x（通过 corepack 安装）
- Docker Desktop / Docker Engine（用于本地 PostgreSQL、可选）
- Git 2.40+

## 2. 初始化步骤
1. 克隆仓库
   ```bash
   git clone git@github.com:your-org/ai-code-review.git
   cd ai-code-review
   ```
2. 安装 Node.js 与 pnpm
   ```bash
   nvm install 20
   nvm use 20
   corepack enable
   corepack prepare pnpm@9 --activate
   ```
3. 安装依赖
   ```bash
   pnpm install
   ```

## 3. 环境变量
1. 复制示例文件
   ```bash
   cp .env.example .env.local
   ```
2. 在 `.env.local` 中填写以下字段（凭证需通过公司 KMS/安全服务申请）：
   - `GITLAB_TOKEN`
   - `GITHUB_APP_ID` / `GITHUB_PRIVATE_KEY`
   - `DINGTALK_WEBHOOK`
   - `LLM_API_KEY`
   - `DATABASE_URL`（本地开发可使用 `postgres://ai-review:ai-review@localhost:5432/ai_review`）
   - 其他内部服务密钥视需要补充
3. 如使用本地 PostgreSQL，可在 `docker-compose.yaml` 中启用数据库服务，并将 `DATABASE_URL` 指向该实例。

## 4. 本地服务启动
1. （可选）启动 PostgreSQL
   ```bash
   docker compose up -d postgres
   ```
2. 执行数据库迁移
   ```bash
   pnpm prisma migrate deploy
   ```
3. 启动后台服务与 Web 控制台
   ```bash
   pnpm dev
   ```
4. 默认访问入口
   - 审查服务 API: `http://localhost:4000`
   - 管理控制台: `http://localhost:3000`

## 5. 测试与质量检查
- 代码风格：`pnpm lint`
- 单元测试：`pnpm test`
- 集成测试：`pnpm test:integration`（需模拟 GitLab/GitHub webhook）
- 预提交检查：执行 `pnpm lint && pnpm test` 确保无失败

## 6. 调试与常见问题
- 使用 `pnpm dev --inspect` 启动审查服务，可配合 VS Code Debugger。
- 若 webhook 调试失败，确认公网可达并检查平台凭证权限。
- 若数据库连接报错，确认 `docker compose` 已启动且 `.env.local` 配置正确。

## 7. 安全注意事项
- 所有密钥必须通过 KMS 注入或本地安全凭证管理工具，严禁提交到仓库。
- 使用独立的测试仓库与机器人账号触发 GitLab/GitHub 事件，避免影响生产仓库。
- 如需抓取真实 MR 数据，确保仅使用只读凭证，并遵守团队合规政策。

## 8. 后续文档
- 部署流程详见 `docs/deployment-playbook.md`（编写中）。
- 常见故障排查建议在 `docs/troubleshooting.md` 补充（待创建）。
