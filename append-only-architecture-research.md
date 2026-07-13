# “只做新增、不做修改”的软件架构调研

> 这里的“只做新增、不做修改”指 **append-only / immutable-history**：变化通过追加事实、版本或数据块表达，而不是覆盖既有历史。它不是一个单独的架构模式，也不意味着数据永不物理删除。

## 摘要

这类架构的共同结构是：

1. **历史记录只追加**：写入新事件、新版本、新对象或新数据块；
2. **当前状态是派生结果**：通过重放、投影、索引、最新指针或可达引用得到；
3. **修改由新事实表达**：追加修正事件或新版本，不回写旧事实；
4. **删除先逻辑生效**：追加 tombstone、delete marker、撤销事件或关闭有效期；
5. **物理回收异步进行**：由 compaction、segment cleaning、GC 或保留策略完成。

它能提供良好的审计、追溯、重建、时间旅行和顺序写特性，但会把复杂度转移到读取、投影一致性、模式演化、容量治理和垃圾回收。工程上通常必须同时建设物化视图、快照、幂等消费、压缩/回收以及明确的保留制度。

## 1. 概念边界

“不可变”至少有四种不同强度，设计评审时不应混为一谈：

| 层级 | 含义 | 例子 |
|---|---|---|
| 逻辑追加 | 应用写路径不更新历史，以后续记录改变语义 | Event Sourcing、tombstone |
| 历史留痕 | 当前值仍可更新，但旧版本自动保存 | SQL Server temporal table |
| 版本级保留 | 单个版本在期限内不可删除或覆盖 | S3 Object Lock |
| 合规 WORM | 在规定条件下连高权限主体也不能改删受保护版本 | S3 Object Lock Compliance mode 等 |

因此：

- **append-only 不等于永久保存**；
- **保存历史不等于 WORM**；
- **对象不可变不等于名称不可复用**；
- **逻辑不修改不等于存储介质从不重写**，压缩和回收经常会搬移存活数据。

## 2. 统一抽象

可把这类系统抽象成以下数据流：

```text
Command
   │ 校验、授权、并发检查
   ▼
Append immutable fact/version ──────► Authoritative history
   │                                      │
   │ publish/CDC                          │ snapshot / retention
   ▼                                      ▼
Projectors ──► Read models / indexes   Compaction / GC
   │
   ▼
Queries return interpreted “current state”
```

历史可以表示为：

\[
H_n = [e_1,e_2,\ldots,e_n]
\]

当前状态不是被直接覆盖的单个值，而是历史的折叠：

\[
S_n = fold(S_0,H_n)
\]

加入快照后：

\[
S_n = fold(S_k,[e_{k+1},\ldots,e_n])
\]

其中快照只是加速结构；若把历史作为事实来源，快照不应成为唯一权威记录。

## 3. 主要架构形态

### 3.1 Append-only log

最基础的形式是将变化按某个顺序追加到日志。日志可作为：

- 事实记录；
- 系统间复制或订阅的接口；
- 状态机重放输入；
- 审计轨迹。

关键不是“文件只能在末尾写”，而是已提交记录具有稳定身份和顺序，消费者用 offset、序列号或版本号跟踪进度。

**常见配套**：分段日志、校验和、幂等 producer/consumer、按 key 分区、保留期、压缩日志、消费位点。

### 3.2 Event Sourcing

Event Sourcing 把每次状态变化保存为不可变事件，事件存储是事实来源，当前状态由事件按序重放得到。[AWS][aws-es] 与 [Microsoft][ms-es] 都强调按时间顺序保存不可变事件以及重建状态。

示例：

```text
AccountOpened(id=A, balance=0)
MoneyDeposited(id=A, amount=100)
MoneyWithdrawn(id=A, amount=30)
WithdrawalReversed(id=A, amount=30, reason=...)
```

这里不会把 `balance=70` 原地改回 `100`；系统追加一个具有业务含义的反转事件。

**设计要求**：

- 事件用过去时、表达已经发生的业务事实；
- 每个 aggregate stream 维护预期版本，采用乐观并发控制；
- 投影器必须可以幂等重试；
- 外部副作用不能因重放而重复发生；
- 事件模式是长期契约，不能像普通 DTO 一样随意修改。

### 3.3 CQRS

CQRS 分离命令模型和查询模型，让两者独立优化；它**不要求** Event Sourcing 或消息系统。两者组合时，常见结构是事件存储承担写模型，事件驱动一个或多个物化读模型。[Microsoft CQRS][ms-cqrs]

收益包括读写独立扩缩容、查询模型反规范化、不同读模型服务不同场景。代价是：

- 读侧通常最终一致；
- 写数据库与发布消息存在原子性缺口；
- 重建、回放和版本迁移更复杂；
- 用户可能短暂读不到刚完成的写入。

实际系统通常用 transactional outbox 或数据库 CDC 缩小双写缺口，并通过事件 ID、业务键和 checkpoint 实现幂等消费。

### 3.4 Log-structured storage

日志结构存储将新数据顺序写入新位置，而不是就地覆写旧块。经典 Sprite LFS 论文提出 segment cleaning：识别仍存活的数据、搬移它们并回收旧段。[LFS 论文][lfs]

这说明“只新增”往往只是**前台写入模型**。后台仍需重写和回收：

```text
旧段: [dead][live][dead][live]
                    │ cleaning
                    ▼
新段: [live][live]        旧段释放
```

低利用率有利于清理性能但浪费容量，高利用率节省空间却增加写放大。这一权衡同样存在于 LSM-tree、SSTable 和许多对象/日志存储中。

### 3.5 Tombstone：用新增表达删除

Cassandra 不直接在所有副本中立即擦除旧值，而是写入带时间戳的 tombstone；读取时 tombstone 屏蔽更旧数据，满足宽限期、repair 和 compaction 等条件后才可能被清除。[Cassandra][cassandra-tombstone]

此机制揭示了分布式删除的难点：如果墓碑被过早清除，离线副本中的旧值可能在 repair 时“复活”。所以删除也必须服从因果顺序、复制延迟和故障窗口。

### 3.6 Temporal 与 bitemporal 数据

SQL Server system-versioned temporal table 维护：

- 当前表中的当前行；
- 历史表中的旧行版本；
- 由数据库管理的系统有效时间区间。

UPDATE 会产生旧版本历史，DELETE 会让当前行退出并保留关闭时间的历史版本。[SQL Server Temporal][sql-temporal]

这属于“当前可变、历史留存”，不是严格 WORM。

完整的 **bitemporal** 模型还会同时记录：

1. **valid time**：事实在业务世界何时有效；
2. **system/transaction time**：系统何时得知并记录该事实。

它可以回答“按当时掌握的信息，我们认为某天的事实是什么”，适合迟到数据、追溯更正、财务和监管场景，但区间约束、查询和修订语义明显更复杂。

### 3.7 Git 与内容寻址存储

Git 对象以内容摘要标识；内容变化会生成新对象，commit 形成指向 tree 和父 commit 的不可变历史。分支只是可移动引用，因此“更新分支”本质上是创建新对象后移动引用。

删除通常意味着移除引用或使对象不可达，而非立即擦除。不可达对象最终可由 reflog 过期和 `git gc` 回收；并发写入期间进行激进裁剪存在损坏风险。[Git GC][git-gc] [Git cruft packs][git-cruft]

因此 Git 同时展示了两个原则：

- 内容不可变与可变引用可以共存；
- 不可达不等于已经物理删除。

### 3.8 WORM 与 S3 Object Lock

S3 Object Lock 在**对象版本**级别提供 WORM 保留和 legal hold，前提是启用 Versioning。受保护版本不能在保留条件结束前被覆盖或删除，但同一个 key 仍可创建新版本或添加 delete marker。[S3 Object Lock][s3-lock]

应区分：

- **Governance mode**：具有特殊权限的主体可以绕过；
- **Compliance mode**：保留期内提供更强的不可删除保证；
- **Legal hold**：没有固定到期日，但可由有权限主体解除。

所以 WORM 的保护对象、权限边界和期限必须写进威胁模型，不能只写“不可变”。

### 3.9 Immutable Infrastructure

不可变基础设施把服务器、镜像或部署单元视为一次性版本：升级不登录机器原地修改，而是构建新镜像、部署新实例、切换流量，再销毁旧实例。

它和不可变数据共享“以替换代替修改”的思想，但目的不同：

- 数据架构强调历史与事实；
- 基础设施强调环境一致性、可重复部署和减少配置漂移。

旧实例最终会删除，因此它通常不是 append-only 的永久历史系统；制品仓库、部署记录和审计日志才可能承担历史追踪。

## 4. 如何表达 CRUD

| 传统操作 | 追加式表达 |
|---|---|
| Create | 追加创建事件或对象版本 |
| Read | 折叠事件、读取投影、沿最新指针或引用读取 |
| Update | 追加新版本、修正/替代事件、移动当前指针 |
| Delete | 追加撤销事件、tombstone、delete marker、关闭有效期或移除引用 |
| Undelete | 追加恢复事件、移除 marker，或重新建立引用（取决于保留状态） |
| Physical erase | 在保留和安全条件满足后执行 compaction、GC 或加密擦除 |

### 修正错误历史

应区分两类情况：

1. **过去的事实确实发生过，之后被业务反转**：追加补偿/反转事件；
2. **记录本身错误或含敏感信息**：需要受控修订、加密擦除或特殊管理流程，不能假装追加一个“忽略它”事件就满足删除义务。

不可变架构不应成为拒绝隐私删除、安全清除或纠错的借口。

## 5. 一致性与并发

### 5.1 乐观并发

对某条事件流使用期望版本：

```text
read stream A at version 12
compute events
append events only if current version == 12
otherwise reject and retry/reconcile
```

这可防止静默覆盖，但不能自动解决跨 aggregate 或跨服务事务。

### 5.2 顺序边界

全局总顺序通常昂贵且不必要。应明确真正需要顺序的边界，例如：

- 单个账户；
- 单个订单；
- 单个分区键；
- 某个合规日志链。

跨边界流程可用 saga/process manager、幂等命令和补偿处理。

### 5.3 投影一致性

读模型消费事件时应保存：

- 事件唯一 ID；
- stream/partition offset；
- 投影版本；
- 最后成功 checkpoint。

投影更新和 checkpoint 最好在同一事务中；否则必须允许安全重复。对“写后读”需求，可返回写入版本并等待读模型达到该版本，而不是声称系统始终强一致。

### 5.4 事件发布原子性

数据库写入与消息发布通常无法共享一个分布式事务。[Microsoft CQRS][ms-cqrs] 明确指出这一一致性挑战。常见方案：

- event store 自身作为订阅源；
- transactional outbox；
- 数据库 CDC；
- 幂等消费者和重复投递；
- dead-letter 与可观测的重试流程。

## 6. 读取、快照和生命周期治理

### 6.1 物化视图

不要让在线查询每次重放全部历史。为查询建立可丢弃、可重建的投影：

- 当前状态表；
- 搜索索引；
- 统计聚合；
- 按用户或场景定制的反规范化视图。

权威历史与缓存/投影必须被清楚区分。

### 6.2 快照

快照减少长事件流的重放成本，但会引入：

- 快照格式版本；
- 快照与事件 offset 的对应关系；
- 校验与损坏恢复；
- 生成快照期间的并发处理。

快照频率应由恢复时间目标、重放吞吐和事件体积决定，而不是固定“每 N 条”后不再测量。

### 6.3 Compaction 与 GC

逻辑删除后，物理空间回收可能包括：

- 合并段/SSTable；
- 搬移存活记录；
- 删除不可达对象；
- 丢弃已过保留期的事件；
- 对敏感数据实施加密擦除。

清理需要安全宽限期，且必须理解副本、备份、归档和下游投影是否仍引用旧数据。

### 6.4 保留策略

建议按数据类别明确：

| 项目 | 需要回答的问题 |
|---|---|
| 最短保留期 | 审计或业务恢复至少需要多久？ |
| 最长保留期 | 隐私和监管允许保存多久？ |
| Legal hold | 谁能设置/解除，如何审计？ |
| 删除范围 | 主存、投影、缓存、备份、导出是否同步处理？ |
| 回收条件 | 时间、可达性、repair、确认消费等哪些条件必须满足？ |
| 删除证明 | 如何验证目标数据已不可恢复？ |

## 7. 模式演化

历史事件会被多年后的代码读取，因此模式演化是核心架构问题。

常见策略：

1. **宽容读取**：新增可选字段，旧读者忽略未知字段；
2. **Upcaster**：读取时把旧事件转换为当前内存形态，不改历史；
3. **事件版本化**：并存 `EventV1`、`EventV2`，显式维护转换；
4. **补充事件**：新增事实补足过去缺失的信息；
5. **重建投影**：在新投影版本中重新解释历史；
6. **受控迁移**：极少数场景重写历史，但必须保留来源、校验和与审计链，并承认系统不再是严格不可变。

应避免：改变既有字段语义、复用事件名称表达不同事实、删除重放所需的旧代码却不提供 upcaster。

## 8. 优势、代价与适用边界

### 优势

- 完整审计与责任追踪；
- 可重建任意历史时点状态；
- 可创建新读模型而不改变写模型；
- 顺序写通常更适合存储介质和复制；
- 调试时能观察状态如何形成；
- 内容寻址可提供去重和完整性校验；
- 与异步集成、流处理天然契合。

### 代价

- 存储持续增长，必须治理保留与回收；
- 在线读取需要投影、索引或快照；
- 最终一致性进入用户体验和业务流程；
- 事件契约及重放代码需要长期维护；
- 删除、隐私合规和敏感数据泄漏更难处理；
- compaction、GC、repair 带来写放大和运维风险；
- 外部副作用无法简单重放；
- 人员需要理解时间、顺序、幂等及可达性语义。

### 适合

- 账务、订单、库存、权限变更等需要审计的领域；
- 状态演化本身具有业务价值；
- 需要历史查询、纠纷复盘或时间旅行；
- 多种读模型及异步下游较多；
- 写入天然是连续事实流；
- 合规归档或版本级 WORM；
- 构建制品、部署和内容寻址对象。

### 不适合或应谨慎

- 简单 CRUD 足以表达且历史没有价值；
- 团队无法长期维护事件契约和投影基础设施；
- 业务要求跨大量实体的同步强一致事务；
- 数据必须频繁彻底删除，且无法通过分层或加密擦除解决；
- 历史中容易写入凭证、密钥或高敏感明文；
- 只因“性能听起来更好”而采用，却没有测量清理和读放大成本。

## 9. 推荐的落地方法

### 第一步：选择不可变边界

不要把整个系统一次性事件化。先选一个审计价值高、聚合边界清楚的领域，例如账户余额或订单生命周期。

### 第二步：定义事实与不变量

- 哪些事件是业务事实？
- 每条 stream 的身份是什么？
- 哪些约束在写入前同步校验？
- 需要局部顺序还是全局顺序？

### 第三步：先设计删除和合规

在上线前明确 tombstone、保留期、备份删除、敏感字段、legal hold 和物理回收，不要等数据已不可控增长后补设计。

### 第四步：把投影当成正式子系统

提供幂等、checkpoint、版本化、重建、延迟监控和失败隔离。投影“可重建”必须通过演练证明，而不是文档假设。

### 第五步：制定事件兼容规则

建立 schema registry 或等价流程；在 CI 中用历史样本测试新代码是否仍能反序列化和重放旧事件。

### 第六步：量化运行指标

至少监控：

- append 延迟与失败率；
- stream 冲突率；
- 投影 lag 和重试次数；
- 重放吞吐与完整重建耗时；
- tombstone/无效数据比例；
- compaction/GC 写放大；
- 历史、快照、投影和备份容量；
- 删除请求从逻辑生效到物理清除的时长。

## 10. 常见误区

1. **“用了消息队列就是 Event Sourcing”**：消息传输不等于把事件作为事实来源。
2. **“CQRS 必须有两套数据库”**：核心是模型分离，物理存储分离是可选深化。
3. **“append-only 自动解决并发”**：它保存冲突事实，但仍需版本校验和业务仲裁。
4. **“不可变就不能删”**：逻辑不可变与依法删除必须通过生命周期设计协调。
5. **“快照可以替代事件”**：若快照成为唯一来源，就失去了历史重建能力。
6. **“投影总能重建”**：外部依赖、旧 schema、已删除代码和非确定逻辑都可能使重建失败。
7. **“日志结构只有顺序写收益”**：清理、写放大、空间利用率和读放大同样决定效果。
8. **“Git 对象永久存在”**：对象不可变，但失去引用后可被 GC。
9. **“Temporal table 就是 bitemporal”**：只有系统时间并不足以表达业务有效时间。
10. **“WORM 保护整个 key”**：S3 Object Lock 保护具体版本，新版本和 delete marker 仍可创建。

## 11. 架构评审清单

- [ ] 权威事实源是什么？投影和缓存能否明确识别？
- [ ] 不可变发生在哪一层，保证强度如何？
- [ ] 顺序和原子性的边界是什么？
- [ ] 更新、更正、撤销和删除分别如何表达？
- [ ] 是否存在写库与发布消息的双写窗口？
- [ ] 消费者是否幂等，是否保存 checkpoint？
- [ ] 是否能在隔离环境完整重建投影？
- [ ] 快照是否版本化并可校验？
- [ ] 事件 schema 如何向前/向后兼容？
- [ ] compaction、repair 和 GC 的安全条件是什么？
- [ ] 保留策略是否覆盖副本、备份、缓存和导出？
- [ ] 如何处理隐私删除、legal hold 和加密擦除？
- [ ] 是否测量读放大、写放大、容量与恢复时间？
- [ ] 采用此模式的业务收益是否高于长期复杂度？

## 12. 结论

“只做新增、不做修改”的真正价值，不在于禁止某个 SQL `UPDATE`，而在于将**事实历史**和**当前解释**分离：历史负责可追溯，投影负责可用性，回收机制负责有限资源，保留制度负责法律和治理边界。

成熟设计通常不是“任何东西永远只能追加”，而是：

> 在需要审计和重建的边界内追加不可变事实；以版本、投影和引用表达当前状态；以 tombstone 或撤销事实表达删除；最后在明确的安全与合规条件下回收物理数据。

如果没有投影、快照、模式演化、幂等、一致性和生命周期治理，append-only 只会把一次简单更新变成无限增长的日志；这些配套齐全时，它才是一种可持续的软件架构。

## 参考资料

以下优先列出官方文档与原始论文。产品默认值和行为可能随版本变化，实施时应固定并核对具体版本。

1. AWS Prescriptive Guidance, [Event sourcing pattern][aws-es].
2. Microsoft Azure Architecture Center, [Event Sourcing pattern][ms-es].
3. Microsoft Azure Architecture Center, [CQRS pattern][ms-cqrs].
4. Mendel Rosenblum, John K. Ousterhout, [The Design and Implementation of a Log-Structured File System][lfs], *ACM TOCS*, 1992.
5. Apache Cassandra Documentation, [Tombstones][cassandra-tombstone].
6. Microsoft Learn, [Temporal tables][sql-temporal].
7. AWS S3 User Guide, [Locking objects with Object Lock][s3-lock].
8. Git Documentation, [git-gc][git-gc].
9. Git Documentation, [Cruft packs][git-cruft].
10. Martin Fowler, [Event Sourcing][fowler-es]（经典概念说明，非产品规范）。

[aws-es]: https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/event-sourcing.html
[ms-es]: https://learn.microsoft.com/en-us/azure/architecture/patterns/event-sourcing
[ms-cqrs]: https://learn.microsoft.com/en-us/azure/architecture/patterns/cqrs
[lfs]: https://people.eecs.berkeley.edu/~brewer/cs262/LFS.pdf
[cassandra-tombstone]: https://cassandra.apache.org/doc/latest/cassandra/managing/operating/compaction/tombstones.html
[sql-temporal]: https://learn.microsoft.com/en-us/sql/relational-databases/tables/temporal-tables?view=sql-server-ver17
[s3-lock]: https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lock.html
[git-gc]: https://git-scm.com/docs/git-gc
[git-cruft]: https://git-scm.com/docs/cruft-packs
[fowler-es]: https://martinfowler.com/eaaDev/EventSourcing.html
