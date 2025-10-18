**Session Date:** 2025-10-18
**Facilitator:** Business Analyst Mary
**Participant:** 用户

## Executive Summary

**Topic:** 对一个项目进行 AI 代码审查

**Session Goals:** 聚焦设计一套可落地的 AI 代码审查方案，强化即时反馈与风险治理

**Techniques Used:** 角色扮演、第一性原理拆解、形态分析

**Total Ideas Generated:** 11

### Key Themes Identified:
- AI 审查需在 MR 阶段提供可信的即时建议，覆盖所有问题类型
- 严重问题必须自动阻断合并并 @ 责任人，以确保及时整改
- 管理层需要按周期查看风险趋势与责任分布，掌握质量脉搏

## Technique Sessions

### 角色扮演（Role Playing） - 约20分钟
**Description:** 轮流站在开发者、审查者、工程经理三个角色视角，挖掘各自诉求与痛点。

#### Ideas Generated:
1. 开发者希望提交后立即获得明确的修改建议，且不排斥 AI 全程介入。
2. 审查者期望 AI 在 push/MR 后自动分析全部风险类型，并在严重问题上 @ 提交者。
3. 工程经理期待按周/月输出风险数量趋势，并关联到具体提交者以便治理。
4. 工程经理要求严重问题未及时处理时，能在钉钉群自动提醒并 @ 指定人员。

#### Insights Discovered:
- 不同角色都强调“即时性”与“可追责性”，说明反馈链路必须统一。
- 对 AI 审查的信任基于可见证据与明确行动指引。

#### Notable Connections:
- @ 提交者与钉钉通知可串联成统一的升级流程。
- 角色视角之间形成了从“发现问题”到“追踪解决”的闭环需求。

### 第一性原理拆解（First Principles Thinking） - 约25分钟
**Description:** 回到代码审查的根本目标与必要要素，从基础假设推导最小可行流程。

#### Ideas Generated:
1. 审查核心是提前规避上线风险，必须掌握改动上下文与业务意图。
2. AI 需要读取改动文件及关键关联文件，理解语义才能定位问题。
3. 最小流程包括：获取改动 → 理解意图 → 风险扫描 → 生成证据化结论 → 给出修改建议 → 反馈与跟踪。
4. 人类审查者主要在合并前查看 AI 结论，判断是否继续推进。

#### Insights Discovered:
- “可靠观点 + 修改建议 + 证据”构成 AI 审查的信任三件套。
- 上下文深度需要控制在“一到两层调用链”，兼顾准确与性能。

#### Notable Connections:
- 将意图理解与风险扫描结合，可为后续评论分级提供依据。
- 合并前的人工复核与 AI 评论可以共享同一平台流程。

### 形态分析（Morphological Analysis） - 约30分钟
**Description:** 定义关键参数及取值，组合出最契合需求的方案，并细化实现细节。

#### Ideas Generated:
1. 选定“Merge Request 创建时触发 + 关联依赖上下文”的组合，强化合并前把关。
2. 检测范围聚焦逻辑异常与测试相关提醒，辅以风格检查以提升可读性。
3. 输出直接写入 MR/PR 评论，并按等级分为阻断、警告、提示、确认。
4. 严重问题在评论中声明阻断合并，并自动 @ 提交者或负责人。
5. 报表采用按月统计风险趋势与提交者分布，支持管理层决策。

#### Insights Discovered:
- 不额外生成 Issue，可避免流程割裂，降低维护成本。
- 等级化评论模板使审查者能快速筛选重点，提升处理效率。

#### Notable Connections:
- 评论等级与升级策略相互匹配，形成清晰的风险处理路径。
- 月度报表与钉钉提醒组合，兼顾长期趋势与即时响应。

## Idea Categorization

### Immediate Opportunities
*Ideas ready to implement now*
1. **MR 评论等级模板落地**
   - Description: 在 MR/PR 中使用统一的“[阻断]/[警告]/[提示]/[确认]”评论格式，并附带代码引用。
   - Why immediate: 可立即提升信息清晰度与审查效率。
   - Resources needed: 定义模板、在审查机器人中实现评论格式。

2. **合并前自动 AI 审查流程**
   - Description: 在 MR 创建时触发 AI 分析，读取改动文件及一至两层关联依赖。
   - Why immediate: 直接解决目前缺乏审查自动化的问题。
   - Resources needed: CI/CD 集成、调用模型与静态分析工具。

### Future Innovations
*Ideas requiring development/research*
1. **风险趋势可视化仪表板**
   - Description: 将阻断/警告级别统计按月展示，并关联提交者。
   - Development needed: 数据采集、报表生成、权限控制。
   - Timeline: 中期（约 1-2 个月）。

2. **钉钉群自动升级机制**
   - Description: 未在时限内处理的阻断问题自动推送钉钉提醒并 @ 责任人。
   - Development needed: 与钉钉机器人集成、设定阈值与逻辑。
   - Timeline: 中期（约 1 个月）。

### Moonshots
*Ambitious, transformative concepts*
1. **跨项目学习的自进化审查引擎**
   - Description: 基于历史审查数据和团队反馈持续训练模型，自动适配不同项目风格。
   - Transformative potential: 可大幅提升审查准确率与团队信任度。
   - Challenges to overcome: 数据隐私、安全合规、模型持续训练成本。

### Insights & Learnings
*Key realizations from the session*
- 即时性、证据化与可追责是 AI 审查成功的三大支柱。
- 结合调用链扩展的一到两层上下文能显著提升分析质量，同时控制性能消耗。
- 统一的评论等级规范是连接自动化审查与人工决策的关键纽带。

## Action Planning

### #1 Priority: MR 评论等级模板落地
- Rationale: 提升沟通效率，让严重问题一目了然。
- Next steps: 设计评论格式 → 在审查机器人中实现 → 与团队对齐使用规范。
- Resources needed: 工程实现、模板设计、团队培训。
- Timeline: 2 周内完成设计与初版部署。

### #2 Priority: 合并前自动 AI 审查流程
- Rationale: 在关键关口阻挡问题，满足开发者与审查者需求。
- Next steps: 配置触发条件 → 对接分析引擎 → 验证评论输出。
- Resources needed: DevOps 支持、模型调用资源、测试用例。
- Timeline: 4 周内完成 MVP。

### #3 Priority: 钉钉群升级提醒机制
- Rationale: 确保严重问题未被忽视，符合管理者诉求。
- Next steps: 定义升级阈值 → 接入钉钉机器人 → 试运行并调整。
- Resources needed: 钉钉 webhook、脚本开发、运维协作。
- Timeline: 6 周内上线试点。

## Reflection & Follow-up

### What Worked Well
- 角色扮演快速统一多角色需求。
- 第一性原理拆解帮助明确最小可行流程。
- 形态分析让方案参数化，便于后续落地。

### Areas for Further Exploration
- 扩展 AI 对业务需求文档的理解能力：评估接入需求管理系统。
- 评估模型推理成本：需要测算运行时长与资源使用以控制预算。

### Recommended Follow-up Techniques
- SCAMPER：用于迭代评论模板和升级流程。
- Mind Mapping：梳理完整的风险类型与应对策略映射。

### Questions That Emerged
- 如何量化 AI 审查的准确率与误报率？
- 模型输出的建议需要怎样的人工反馈机制来持续优化？

### Next Session Planning
- **Suggested topics:** 评估技术栈与工具选型、定义审查成功指标
- **Recommended timeframe:** 2-3 周后，待初版流程设计完成
- **Preparation needed:** MVP 方案草案、可行性评估、初步成本数据

---

*Session facilitated using the BMAD-METHOD™ brainstorming framework*
