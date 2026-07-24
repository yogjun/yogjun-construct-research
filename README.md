# yogjun-construct-research

## 调研文档

- [项目代码“只增整删、不做行级修改”的架构调研](./append-only-architecture-research.md) — 研究已有文件内容冻结、只新增或整删文件/模块的迭代约束，覆盖插件发现、版本化替换、外围演进、安全修复和 CI 门禁。
- [带最早执行时间与活跃 Key 去重的任务队列设计](./scheduled-deduplicated-task-queue-design.md) — 面向每小时约 12,000 条任务，设计基于 MongoDB 任务集合、活跃 key 部分唯一索引和多实例原子领取的简单队列方案。
- [AI Agent 开发框架技术选型调研（2026 全景）](./Agent框架选型/ai-agent-framework-selection-research.md) — 以"控制流哲学"（确定性图 / Agent 循环 / 多智能体 / 企业内核）为主轴，横切 8 维能力矩阵，深挖 7 个标杆框架；给出复杂度分级、一票否决、选型决策树、生产参考架构、两阶段 PoC 门禁与 2026 趋势，并覆盖 MCP/A2A/AG-UI 协议与国内模型生态。
- [Agent 框架分层与选型 QA 总结](./Agent框架选型/Agent框架分层与选型QA总结.md) — 汇总 Agent 循环、L1/L2 分层、LangChain Agent 与 LangGraph 的依赖关系，以及 Spring AI、LangChain4j 和最小 AI 应用方案的边界。
- [Agent 免中途确认与无人值守执行方案调研](./agent-unattended-execution-research.md) — 区分权限确认、业务澄清和最终审查，比较 Codex、Claude Code、Gemini CLI、OpenHands、Cline、Roo Code、GitHub Copilot cloud agent 与 Aider；给出预授权、自动审查、沙箱、durable execution 和最终 PR 门禁组成的无人值守架构。
- [ReAct 循环中的预期结果验证设计](./Agent框架选型/ReAct循环中的预期结果验证设计.md) — 将 Agent 的“反思”收敛为带证据的预期结果验证协议，覆盖验证契约、三态判定、Verifier 路由、定向重试、两级验证、循环预算和 Judge 校准。
