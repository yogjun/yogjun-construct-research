<!-- cSpell:words Cockburn Seemann -->

# 项目代码“只增整删、不做行级修改”的架构调研

> 本文讨论软件项目迭代中的代码演进约束。这里的“追加”不是向已有函数追加几行代码，也不是数据 append-only，而是：**已有文件内容冻结；需求通过新增完整文件或完整模块实现；退役时允许删除完整文件或完整模块。**

## 摘要

本文采用比一般开闭原则更严格的工程规则：

> **只允许新增完整文件或模块，以及删除完整文件或模块；不允许对已有文件做任何行级调整。**

因此，以下操作均不允许：

- 在已有函数中追加几行代码；
- 修改已有条件、参数、返回值或异常处理；
- 在已有类中增加方法、字段、注解或接口实现；
- 修改已有 import、路由表、注册表、composition root 或配置文件；
- 在原文件中修 bug、升级调用方式或插入 feature toggle；
- 以“先删除旧文件、再用同一路径写入新内容”规避限制。

允许的操作只有：

1. 新增一个完整文件；
2. 新增一个完整目录、包、插件、模块、服务或版本；
3. 删除一个完整文件；
4. 删除一个完整模块及其专属文件；
5. 通过**预先存在且无需改动旧文件**的发现、装配、路由或版本选择机制，让新增单元生效。

这可以称为：

## 文件级不可变演进：只增整删

它比 OCP 的“扩展优先”严格得多。要长期执行，系统必须从一开始就提供基于目录、插件清单、服务发现、约定命名、外部控制面或构建生成物的扩展机制。若旧系统没有这样的扩展缝，新文件即使编写完成也可能永远不可达；在不修改旧文件的前提下，只能在系统外围新增入口/代理，或整模块替换。

## 1. 规则的精确定义

### 1.1 变更单位

规则保护的最小单位是**文件**，推荐治理单位是**模块**。

设一个版本的受控源码文件集合为：

```text
Vn = {f1, f2, ..., fk}
```

下一个版本只能由集合运算产生：

```text
Vn+1 = (Vn - D) ∪ A
```

其中：

- `A` 是全新的文件集合；
- `D` 是完整删除的旧文件集合；
- 对所有同时存在于 `Vn` 和 `Vn+1` 的文件，其内容哈希必须完全相同。

形式化约束：

```text
∀ f ∈ (Vn ∩ Vn+1): hash(Vn[f]) == hash(Vn+1[f])
```

因此，普通 Git `modified` 状态不合规；只允许 `added` 与 `deleted`。

### 1.2 合规操作

| 操作 | 是否允许 | 说明 |
| --- | --- | --- |
| 新增完整源文件 | 允许 | 文件路径此前不存在 |
| 新增完整模块/插件/服务 | 允许 | 包含新增实现、测试和描述文件 |
| 删除完整源文件 | 允许 | 不保留局部残片 |
| 删除完整模块 | 允许 | 删除模块拥有的全部文件和入口 |
| 修改已有函数 | 禁止 | 包括向函数尾部追加代码 |
| 修改已有类 | 禁止 | 包括新增方法、字段、注解和 import |
| 修改已有配置/manifest | 禁止 | 配置文件同样属于冻结文件 |
| 原路径整文件覆盖 | 禁止 | Git 语义仍是修改，不是整删 |
| 删除后重建同路径 | 禁止 | 属于规避约束；应使用新路径/新版本名 |
| 新增 V2 文件并删除 V1 文件 | 允许 | 前提是系统能选择 V2，且删除不会破坏引用 |

### 1.3 文件与模块删除的边界

“允许删除”并不表示可以用删除制造隐式修改：

- 不允许删除共享文件后，把其中部分逻辑复制到多个新文件；
- 不允许删除旧文件并在同一路径提交不同内容；
- 删除模块前必须证明没有存活依赖者；
- 模块删除应同时删除专属测试、配置、资源和文档；
- 公共契约文件若仍有消费者，不得单独删除。

推荐用新身份表示替代版本：

```text
payment-v1/          # 原模块，内容冻结
payment-v2/          # 全新模块
```

迁移完成后，完整删除 `payment-v1/`，而不是在 `payment-v1/` 内逐行改造成 V2。

## 2. 与开闭原则的关系

Robert C. Martin 对 OCP 的经典表述是软件实体应“对扩展开放、对修改关闭”，即系统行为可通过扩展改变，而稳定部分不必反复修改。[Martin 2014][ocp-2014]

但 OCP 本身没有要求“已有文件哈希永远不变”。Martin 后续也强调现实系统只能针对已识别的变化轴进行 strategic closure，而不可能对所有未知变化关闭。[Martin 2013][ocp-2013]

本文规则是在 OCP 之上增加的项目约束：

```text
OCP：尽量通过扩展改变行为。
本文：扩展只能以新增完整文件/模块发生，旧文件不得出现行级变化。
```

所以：

- OCP 是设计原则；
- “只增整删”是变更形态和仓库治理规则；
- 插件化、六边形架构和 DI 可帮助满足规则；
- 这些模式本身并不自动保证旧文件不变；
- 没有预建扩展缝时，严格规则可能迫使系统采用外围替换或整模块升级。

Google 的代码评审准则提醒评审者避免为猜测性的未来需求预建通用机制。[Google Code Review][google-looking] 因此，不应给每个函数都预埋扩展点；更合理的是在模块边界、外部集成、规则系统和协议入口等已知变化面提供少量稳定扩展协议。

## 3. 必需的架构前提

严格执行“旧文件零修改”需要系统在第一次冻结前具备以下能力。

### 3.1 稳定扩展契约

核心必须依赖稳定端口，而不是直接依赖具体实现：

```text
Stable Core ──► Extension Contract
                      ▲
          ┌───────────┴───────────┐
       Plugin V1                Plugin V2
```

契约不仅包括类型签名，还包括：

- 输入输出语义；
- 错误与重试语义；
- 事务和幂等要求；
- 调用顺序与生命周期；
- 性能和容量边界；
- 权限、安全和资源限制；
- 兼容性与弃用规则。

契约文件一旦冻结便不能追加方法。需要新能力时，应新增 `ContractV2` 文件和实现，而不是编辑 `ContractV1`。

### 3.2 无需修改旧文件的发现机制

新增实现必须可以被系统自动发现或由仓库外控制面选择。可选机制包括：

1. 扫描约定目录下的独立插件描述文件；
2. 每个插件自带全新 manifest，而非修改共享 manifest；
3. Java `ServiceLoader` 一类按制品内服务文件发现实现；
4. 模块路径或 classpath 上的服务发现；
5. 服务注册中心与 API gateway 动态路由；
6. Kubernetes label/selector 或同类外部编排机制；
7. 构建时根据目录内容生成**不纳入冻结源码**的聚合注册产物；
8. 内容寻址或版本目录加“当前版本”外部指针。

关键条件是：新增插件时不能去编辑一个共享 `plugins.json`、路由表或容器绑定文件。共享注册表会成为每次变化都必须修改的单点，与规则冲突。

### 3.3 外置 Composition Root

一般 DI 架构允许修改 composition root；本文规则不允许修改已有 composition root 文件。因此必须采用以下一种设计：

- composition root 读取插件目录并动态装配；
- 每个模块携带独立注册贡献文件；
- 构建阶段从新增文件生成对象图；
- 部署控制面选择完整模块版本；
- 新增一个全新的应用入口/启动器，并在部署层切换到它；
- 以新服务替代旧服务，由外部网关切流。

若部署清单也属于冻结仓库，则切换必须由仓库外的部署平台完成；否则只能新增一份全新的版本化部署清单，不能编辑旧清单。

### 3.4 强模块所有权

每个可删除模块应拥有自己的：

```text
module-v2/
├── src/
├── tests/
├── resources/
├── plugin-manifest/
├── migrations/
├── dashboards/
└── docs/
```

共享文件越多，整模块删除越困难。跨模块共享应通过版本化公共包或协议完成，不能通过多个模块共同编辑同一源文件。

## 4. 适配该规则的架构模式

### 4.1 插件与微内核

插件/微内核最接近目标：宿主提供扩展点、发现机制和生命周期，新增能力以独立插件交付。Fowler 的 Plugin 模式把具体实现连接推迟到配置或运行阶段，使宿主不静态依赖具体插件。[Fowler Plugin][fowler-plugin]

IntelliJ Platform 展示了“扩展点 + 插件实现 + 描述元数据”的成熟模式。[IntelliJ Extensions][ij-extensions] [IntelliJ Extension Points][ij-extension-points]

不过 IntelliJ 的标准流程通常需要在插件自己的 `plugin.xml` 中注册扩展。对本文规则而言：

- 新插件连同全新的 `plugin.xml` 一起新增：合规；
- 修改一个已经存在的 `plugin.xml` 增加 extension：不合规；
- 修改宿主以新增 extension point：不合规；
- 新增宿主 V2 或新插件 API V2：合规，但需要外部机制选择它。

### 4.2 Strategy、Decorator、Adapter

这些模式只有在选择机制已预建时才能做到零修改旧文件：

- **Strategy**：新增策略文件，由规则发现机制选择；
- **Decorator**：新增装饰器文件，由声明式 pipeline 自动组装；
- **Adapter**：新增外部系统适配器，由插件协议发现；
- **Handler Chain**：新增 handler 文件，由优先级元数据自动组成链。

如果还需要在已有函数中增加：

```text
if newStrategyEnabled:
    use NewStrategy
```

则不符合本规则。正确方式是新增策略贡献文件，并由冻结的选择器读取；若选择器不存在，只能新增完整 V2 入口或 V2 模块。

### 4.3 六边形架构

Cockburn 的六边形架构通过端口与适配器隔离应用和外部技术。[Cockburn Hexagonal][hexagonal] 它有利于新增适配器，但不天然满足文件不可变：原始示例在首次建立 repository seam 时就需要更新应用以接收 adapter。

因此应用本规则时必须区分：

- **冻结前**：允许建立端口、插件加载器和版本边界；
- **冻结后**：端口 V1 文件不再修改，只能新增适配器或新端口版本；
- **没有端口时**：不得向旧类插入端口，只能新增外围 facade/代理，或新增完整应用 V2。

### 4.4 版本化模块与 Side-by-side

最直接的做法是新旧模块并存：

```text
checkout-v1/   # 冻结
checkout-v2/   # 新增
checkout-v3/   # 将来新增
```

外部路由器、插件选择器或部署控制面决定当前版本。迁移完成后整目录删除 V1。

优点：

- 变更边界清晰；
- 回滚简单；
- 新旧实现可独立测试；
- Git 变更可机械审计。

代价：

- 代码重复可能很高；
- bug/security 修复可能需要生成多个替代版本；
- API 与数据兼容复杂；
- 版本选择和清理成为核心基础设施；
- 共享资源会阻碍整模块删除。

### 4.5 Strangler Fig

当旧应用没有内部扩展缝时，可在外围新增代理、网关或新服务，将新流量导向新模块。旧应用文件保持不变，能力逐步迁出；最后整模块删除旧应用。

严格规则下，路由切换也不能靠修改已有路由文件，可采用：

- 外部 API gateway 控制面；
- 新增版本化路由配置并整体切换；
- 新增一个新的入口服务；
- DNS/负载均衡器的外部发布操作。

### 4.6 Branch by Abstraction 的限制

传统 Branch by Abstraction 需要先修改旧调用方，使其经过抽象，再并行接入新实现。因此它不适用于已经冻结且禁止行级修改的文件。

只有两种情况下可用：

1. 抽象缝在冻结前已经存在；
2. 用一个全新外围模块包住旧模块，而不编辑旧模块内部文件。

### 4.7 Feature Toggle 的限制

传统 feature toggle 常要求在已有函数中插入分支，因此不合规。只有以下形式合规：

- 冻结的路由器原本就支持由外部 flag 选择插件；
- 新旧实现是独立模块；
- flag 配置位于外部控制面，或以全新版本化文件新增；
- 清理时整模块删除旧实现，而不是编辑函数删除 `if` 分支。

Fowler、GitLab 和 LaunchDarkly 都强调 flag 的生命周期和清理。[Fowler Feature Toggles][feature-toggles] [GitLab Feature Flags][gitlab-flags] [LaunchDarkly Flag Debt][ld-flag-debt] 在本文规则下，应避免把 toggle point 写进稳定业务函数，而应把版本选择置于模块边界。

## 5. API、Schema 与数据库边界

### 5.1 公共契约也不得原文件追加

通常 additive API change 会向已有 schema 文件追加字段；本文规则对此仍判定为不合规。应改为：

```text
order-api-v1.proto   # 冻结
order-api-v2.proto   # 新增
```

Protobuf 官方说明新增字段通常可保持 wire compatibility，但 wire-safe 不等于应用语义安全。[Protobuf Updating][protobuf-updating] [Protobuf Best Practices][protobuf-best]

严格文件级演进推荐：

- 已有 schema 文件冻结；
- 新契约使用新文件、新 package 或新 major version；
- 新旧生成代码并存；
- 适配器在模块边界转换；
- 消费者迁移完成后整套删除 V1 文件。

### 5.2 Expand-Contract 的改写

传统 expand-contract 常在同一 schema 文件上先加字段、后删字段，与本规则冲突。应改成**版本单元的 expand-contract**：

```text
Expand
  1. 新增完整 API/Schema V2 文件
  2. 新增 V1↔V2 适配模块
  3. 新增 V2 producer/consumer
  4. 并行部署与验证

Contract
  5. 停止向 V1 路由
  6. 整体删除 V1 模块、V1 schema 和专属适配器
```

不允许在 `v1.proto` 中添加字段，也不允许逐行删除旧字段。

### 5.3 数据库迁移

数据库迁移脚本天然适合新增文件：

```text
001_create_order.sql       # 冻结
002_add_status.sql         # 新增
003_backfill_status.sql    # 新增
```

已执行迁移脚本不得修改。需要回滚或修正时新增补偿迁移文件。这里的规则约束的是**迁移代码文件**，不是要求数据库数据只追加。

如果要删除整套旧数据库模块，应先完成消费者迁移、数据保留和备份治理，再删除拥有该 schema 的完整模块；不能仅删迁移历史以假装旧结构不存在。

## 6. Bug 与安全修复

这是严格规则代价最大的部分。用户给定约束不允许在旧文件中原位修复，因此修复只能采用以下方式：

### 6.1 整文件替代版本

新增具有新身份的修复版本：

```text
auth-handler-v1.ts   # 冻结、有漏洞

auth-handler-v2.ts   # 新增、已修复
```

切换完成后完整删除 V1。不得覆盖 `auth-handler-v1.ts`。

### 6.2 整模块安全升级

对安全敏感模块采用模块版本替换：

```text
auth-v1/  →  auth-v2/
```

V2 必须完整包含修复、测试、权限声明和部署资源。确认所有流量切换后删除 V1，确保漏洞代码不再构建、打包或部署。

### 6.3 外围阻断

若短期无法替换，可新增 WAF 规则模块、网关策略或安全代理阻断攻击路径。但这只能作为临时缓解；只要漏洞旧模块仍可达，就不能视为修复完成。

### 6.4 无法切换时的结论

若系统既不允许修改旧文件，又没有插件、版本路由或模块替换能力，那么某些漏洞将无法修复。此时必须明确二选一：

1. 允许一次例外修改；或
2. 停用并整模块删除受影响能力。

不能为了保持规则而让已知漏洞继续暴露。若用户坚持零例外，则**停用/整删**是唯一合规安全动作。

## 7. 装配、注册和配置

本文的新限制意味着已有装配文件同样冻结。

### 7.1 禁止的方式

```text
// existing-registry.ts
registry.add(NewPlugin)  // 即使只追加一行也禁止
```

以下同样禁止：

- 向已有 `plugin.xml` 追加节点；
- 向已有路由表追加 endpoint；
- 向已有 DI module 追加 binding；
- 向已有构建文件追加 source set；
- 向已有配置增加 feature flag；
- 向已有函数加入自动发现逻辑。

### 7.2 合规方式

推荐每个扩展携带独立贡献文件：

```text
plugins/
├── payment-v1/plugin.yaml
├── payment-v2/plugin.yaml      # 新增
└── refund-v1/plugin.yaml       # 新增
```

冻结的 loader 扫描 `plugins/*/plugin.yaml`。删除插件时整目录删除。

其他方式：

- 新增完整应用入口 `app-v2`；
- 新增完整路由版本 `routes-v2`，由外部平台切换；
- 新增独立部署单元，由服务发现接入；
- 构建时生成临时聚合文件，且生成文件不作为人工维护的冻结源码；
- 使用包管理器/模块系统自带的服务发现机制。

## 8. 分层约束

| 区域 | 新增 | 行级修改 | 整体删除 | 推荐演进方式 |
| --- | --- | --- | --- | --- |
| 稳定核心文件 | 仅新增 V2 | 禁止 | 允许 | 新核心版本或外围替代 |
| 公共契约文件 | 新增 V2 | 禁止 | 消费者清零后允许 | 版本化契约 |
| 插件/适配器 | 允许新增 | 禁止 | 允许 | 新插件版本 |
| composition root | 新增新入口 | 禁止 | 允许 | 动态发现或入口版本化 |
| 注册/路由配置 | 新增独立贡献文件 | 禁止 | 允许 | 目录扫描/外部控制面 |
| 测试 | 新增测试文件 | 禁止 | 随模块整删 | 版本化 contract suite |
| 迁移脚本 | 只新增 | 禁止 | 通常保留审计历史 | forward-only migration |
| 文档 | 新增版本文档 | 禁止 | 允许 | 版本化 ADR/说明 |

### 8.1 项目规范

```text
R1. 已有文件内容冻结；任何 surviving file 的前后哈希必须一致。
R2. 只允许新增全新路径的文件/模块，或删除完整文件/模块。
R3. 禁止向已有函数、类、配置、manifest、路由表或构建文件追加任何行。
R4. 禁止通过删除后同路径重建、重命名往返或格式化来规避修改禁令。
R5. 新能力必须经预建扩展协议、自动发现、外部控制面或新增完整入口接入。
R6. 没有扩展缝时，只能新增外围模块/应用 V2，不能编辑旧调用点。
R7. 公共契约通过新增 V2 文件演进，不在 V1 文件中追加字段或方法。
R8. bug/security fix 通过新增修复版本并整删漏洞版本完成；无法替换时停用整模块。
R9. 新旧版本并存必须有 owner、退出指标、截止日期和整删任务。
R10. 迁移完成后，完整删除旧文件/模块，不做局部清理。
R11. 删除前必须证明无存活引用、无生产流量、无必要数据和无共享所有权。
R12. CI 只接受 Added/Deleted 文件状态；Modified/Renamed-with-edits 一律失败。
```

## 9. 工程门禁

### 9.1 Git 变更门禁

CI 应拒绝所有内容修改：

```text
允许：A path/to/new-file
允许：D path/to/old-file
拒绝：M path/to/existing-file
拒绝：R old -> new（若内容发生变化）
```

仅检查 Git 状态仍可能被规避，建议同时：

1. 读取基线版本和候选版本的文件列表；
2. 对交集文件计算内容哈希；
3. 任一交集文件哈希变化则失败；
4. 检测删除后同路径重建；
5. 对重命名按内容哈希识别；
6. 对生成文件和锁文件定义明确策略；
7. 记录经批准的冻结基线。

伪代码：

```text
baseFiles = files(base)
headFiles = files(head)

for path in intersection(baseFiles, headFiles):
    assert hash(base[path]) == hash(head[path])
```

### 9.2 新文件质量门禁

“文件是新增的”不代表质量合格。新增模块仍须通过：

- 单元测试和 contract tests；
- 依赖方向检查；
- 安全扫描和许可证检查；
- API 兼容矩阵；
- 资源与性能预算；
- 插件权限最小化；
- owner、版本和删除条件检查；
- 重复代码分析。

### 9.3 删除门禁

整文件/整模块删除前验证：

- 没有静态 import 或运行时加载引用；
- 没有生产路由和活动租户；
- 没有其他模块依赖其契约；
- 数据已迁移或按政策保留；
- 告警、dashboard、runbook 和部署资源同步删除；
- 制品和容器不再打包漏洞代码；
- 具备回滚制品，而非依赖代码仍在线可达。

## 10. 测试与发布

### 10.1 Contract Test 版本化

旧 contract test 文件不能追加新案例。因此契约测试也应版本化：

```text
contract-v1/   # 冻结
contract-v2/   # 新增，包含 V2 完整规范
```

每个 V2 实现运行 V2 契约；需要兼容 V1 时，同时运行冻结的 V1 契约。

### 10.2 新旧实现对比

可采用：

- shadow traffic；
- 双跑不双写；
- 输出 diff；
- 金丝雀；
- 分租户路由；
- 离线回放。

对有副作用的操作必须避免重复执行。对比组件也应作为独立新增模块，迁移完成后整模块删除。

### 10.3 标准发布流程

1. 确认冻结核心已有扩展/替换路径；
2. 新增完整 V2 模块及独立测试；
3. 新增独立插件贡献文件或部署单元；
4. 由预建 loader 或外部控制面发现 V2；
5. shadow 或小流量灰度；
6. 验证行为、安全、性能和数据兼容；
7. 全量切换；
8. 等待有限回滚窗口；
9. 整体删除 V1 模块及其专属资产；
10. 验证构建、制品和生产环境不再包含 V1。

## 11. 决策流程

```text
需求能否由冻结系统已有的扩展协议发现？
  ├─ 是
  │   └─ 新增完整插件/模块文件 → 测试 → 外部切换 → 整删旧版本
  └─ 否
      │
      ├─ 能否在系统外围新增代理、新入口或新服务？
      │   ├─ 是 → 新增外围 V2 → 渐进切流 → 整删旧模块
      │   └─ 否
      │
      └─ 严格约束下该需求不可实现
          ├─ 重新建设完整应用/模块 V2；或
          └─ 由治理流程决定是否允许例外
```

在“零例外”模式下，最后一个分支只能选择新建完整 V2 或放弃需求，不能悄悄向旧函数追加几行代码。

## 12. 成本与风险

### 12.1 主要收益

- 旧文件内容稳定，回归面清晰；
- 代码审查容易识别新增和整删边界；
- 模块级回滚与替换直接；
- 迫使系统建立明确协议和所有权；
- 避免在旧函数中持续叠加条件分支；
- 已退役实现可以整模块彻底移除。

### 12.2 主要成本

- 为小改动新增完整文件/版本，重复代码可能显著增加；
- 没有预建扩展点的旧系统很难演进；
- composition root、构建和路由必须高度动态化；
- bug/security fix 无法就地修复，只能版本替换或停用；
- 公共契约频繁产生 V2/V3；
- 测试矩阵和部署单元增多；
- 整模块删除需要严格依赖和数据治理；
- 自动发现机制扩大供应链与插件加载攻击面；
- 过度依赖外围 wrapper 可能隐藏核心缺陷；
- 文件级不可变并不自动保证行为兼容。

Google 的评审原则指出，系统复杂性会由许多小变化累积。[Google Review Standard][google-standard] 本规则虽然消除了旧文件内的增量复杂化，却可能把复杂度转移为文件、模块、版本和路由数量。因此必须同时限制活跃版本数量并强制整删。

## 13. 不允许的规避模式

1. **函数尾追加**：声称“没改原逻辑”，只在函数末尾添加代码。
2. **同路径覆盖**：删除旧文件后以相同路径提交不同内容。
3. **Copy-and-Tweak**：复制整个旧文件，只改几行并长期并存。
4. **Wrapper Tower**：不断新增 wrapper，却从不整删底层版本。
5. **共享注册表修改**：每个新插件都去编辑同一个 registry。
6. **永久版本墓地**：V1、V2、V3 全部在线，没有退出计划。
7. **漏洞外包裹**：新增安全层，但漏洞旧入口仍可达。
8. **碎片化删除**：只删部分旧文件，留下无主配置、测试和资源。
9. **生成物偷渡**：把人工逻辑藏进声称可变的生成文件。
10. **重命名洗白**：重命名旧文件后修改内容，假装是新增文件。

## 14. 架构评审清单

### 文件级约束

- [ ] 本次 diff 是否只包含 Added 与 Deleted？
- [ ] 所有基线与候选版本共有文件的哈希是否相同？
- [ ] 是否存在删除后同路径重建或带修改的重命名？
- [ ] 是否向任何已有函数、类、配置或 manifest 添加了行？

### 可达性与装配

- [ ] 新文件如何在不修改旧文件的情况下被发现？
- [ ] 发现机制是否在冻结前已经存在？
- [ ] 是否需要修改共享 registry、路由表或 composition root？若需要则方案不合规。
- [ ] 能否通过新增完整入口、插件或外围服务实现？

### 契约与兼容

- [ ] 新版本是否使用独立契约文件和版本身份？
- [ ] 新旧实现是否通过对应 contract suite？
- [ ] 错误、事务、时序、性能和安全语义是否兼容？
- [ ] API、数据和消息是否支持并行版本？

### 删除

- [ ] 被删文件是否属于可独立退役的完整单元？
- [ ] 是否还有静态、动态或生产流量引用？
- [ ] 专属测试、配置、资源、dashboard 和文档是否一起删除？
- [ ] 删除后是否仍有可靠制品用于有限回滚？
- [ ] 漏洞旧版本是否已从构建与部署环境彻底消失？

### 生命周期

- [ ] 新模块是否有 owner、版本和到期/替换策略？
- [ ] side-by-side 的退出指标和截止日期是什么？
- [ ] 活跃版本数量是否受限？
- [ ] 迁移完成后是否创建了整模块删除任务？

## 15. 结论

本项目的新限制不是一般意义的“尽量不改稳定代码”，而是明确的文件级规则：

> **已有文件零行级修改；新增需求只新增完整文件或模块；退役只删除完整文件或模块。**

这意味着不能向已有函数追加几行代码，也不能修改已有 composition root、注册表、配置、schema 或 feature toggle 分支。新能力必须借助预建自动发现机制、独立插件贡献文件、外部控制面、版本化完整入口或外围新服务生效。

因此，推荐原则为：

## 推荐原则：旧文件冻结、新单元追加、旧单元整删

该原则可以机械验证，但代价和前提必须被接受：

- 系统必须预先具备扩展和版本选择能力；
- 没有扩展缝时需新增完整 V2 或外围替换；
- bug/security fix 只能通过新增修复版本后整删旧版本完成；
- 若漏洞模块无法替换，必须停用整删或批准例外；
- 为防止版本和重复逻辑无限增长，迁移完成后必须整模块删除。

换言之，这是一种**文件/模块级不可变、集合级可增删**的代码演进模型，而不是“可以在旧函数末尾继续追加代码”的行级 append-only 模型。

## 参考资料

1. Robert C. Martin, [The Open Closed Principle][ocp-2014].
2. Robert C. Martin, [An Open and Closed Case][ocp-2013].
3. Martin Fowler, [Plugin][fowler-plugin].
4. JetBrains, [Plugin Extensions][ij-extensions].
5. JetBrains, [Extension Points][ij-extension-points].
6. Alistair Cockburn, [Hexagonal Architecture][hexagonal].
7. AWS Prescriptive Guidance, [Hexagonal architecture pattern][aws-hexagonal].
8. Mark Seemann, [Composition Root][composition-root].
9. Martin Fowler, [Inversion of Control Containers and the Dependency Injection pattern][fowler-injection].
10. Protocol Buffers, [Updating A Message Type][protobuf-updating].
11. Protocol Buffers, [Proto Best Practices][protobuf-best].
12. Martin Fowler, [Feature Toggles][feature-toggles].
13. GitLab, [Feature flags in development][gitlab-flags].
14. LaunchDarkly, [Manage feature flag technical debt][ld-flag-debt].
15. Google Engineering Practices, [What to look for in a code review][google-looking].
16. Google Engineering Practices, [The Standard of Code Review][google-standard].

> 注：这些资料支持 OCP、插件、端口/适配器、依赖注入、契约兼容和迁移治理等基础机制；“已有文件哈希必须不变、只允许整文件增删”是本文在这些机制之上定义的更严格项目约束，并非上述资料共同提出的通用原则。

[ocp-2014]: https://blog.cleancoder.com/uncle-bob/2014/05/12/TheOpenClosedPrinciple.html
[ocp-2013]: https://blog.cleancoder.com/uncle-bob/2013/03/08/AnOpenAndClosedCase.html
[fowler-plugin]: https://martinfowler.com/eaaCatalog/plugin.html
[ij-extensions]: https://plugins.jetbrains.com/docs/intellij/plugin-extensions.html
[ij-extension-points]: https://plugins.jetbrains.com/docs/intellij/plugin-extension-points.html
[hexagonal]: https://alistair.cockburn.us/hexagonal-architecture/
[aws-hexagonal]: https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/hexagonal-architecture.html
[composition-root]: https://blog.ploeh.dk/2011/07/28/CompositionRoot/
[fowler-injection]: https://martinfowler.com/articles/injection.html
[protobuf-updating]: https://protobuf.dev/programming-guides/proto3/#updating
[protobuf-best]: https://protobuf.dev/best-practices/dos-donts/
[feature-toggles]: https://martinfowler.com/articles/feature-toggles.html
[gitlab-flags]: https://docs.gitlab.com/development/feature_flags/
[ld-flag-debt]: https://launchdarkly.com/docs/guides/flags/technical-debt
[google-looking]: https://google.github.io/eng-practices/review/reviewer/looking-for.html
[google-standard]: https://google.github.io/eng-practices/review/reviewer/standard.html
