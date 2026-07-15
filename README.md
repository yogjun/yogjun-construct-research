# yogjun-construct-research

## 调研文档

- [项目代码“只增整删、不做行级修改”的架构调研](./append-only-architecture-research.md) — 研究已有文件内容冻结、只新增或整删文件/模块的迭代约束，覆盖插件发现、版本化替换、外围演进、安全修复和 CI 门禁。
- [带最早执行时间与活跃 Key 去重的任务队列设计](./scheduled-deduplicated-task-queue-design.md) — 面向每小时约 12,000 条任务，设计基于 MySQL 状态表、活跃 key 唯一约束和多实例批量领取的简单队列方案。
