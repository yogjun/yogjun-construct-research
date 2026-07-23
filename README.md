# yogjun-construct-research

## 调研文档

- [项目代码“只增整删、不做行级修改”的架构调研](./append-only-architecture-research.md) — 研究已有文件内容冻结、只新增或整删文件/模块的迭代约束，覆盖插件发现、版本化替换、外围演进、安全修复和 CI 门禁。
- [带最早执行时间与活跃 Key 去重的任务队列设计](./scheduled-deduplicated-task-queue-design.md) — 面向每小时约 12,000 条任务，设计基于 MongoDB 任务集合、活跃 key 部分唯一索引和多实例原子领取的简单队列方案。
- [AI 应用（Agent）开发框架技术选型调研](./ai-agent-development-framework-selection-research.md) — 对比主流 Agent SDK、工作流运行时与 Java/TypeScript/Python/.NET 技术路线，给出分场景选型、生产参考架构、PoC 门禁和迁移策略。
