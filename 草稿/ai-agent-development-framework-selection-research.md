<!-- cSpell:words Agentic AutoGen Bedrock checkpointer CrewAI EvalOps Foundry guardrail handoff LangChain LangGraph LangSmith LlamaIndex Mastra Pydantic Semantic Kernel Strands toolset -->

# AI 应用（Agent）开发框架技术选型调研

> 调研快照：2026-07-22。本文基于各项目官方文档、官方仓库和协议规范形成。Agent 生态变化很快，版本号与语言能力可能在数周内变化；进入项目时仍应锁定版本，并用本文的 PoC 门禁复核。

## 摘要

Agent 框架没有脱离场景的“最佳选择”。真正需要选择的是一组分层能力：模型与工具接入、Agent loop、确定性工作流、有状态运行时、持久化与恢复、人机协作、评测观测、UI 流协议以及部署控制面。把这些能力都寄托在一个框架上，往往会同时产生过度抽象和平台锁定。

本文的核心结论是：

1. **先判断是否真的需要 Agent。** 单次模型调用、RAG 或固定工作流能解决的问题，不应先引入自主 Agent。Agent 用更高的延迟、成本和错误复合风险换取动态决策能力。Anthropic 的生产经验同样建议从最简单的可行方案开始，并明确区分预定义路径的 workflow 与模型自主选择路径的 agent。[Anthropic Effective Agents][anthropic-effective-agents]
2. **业务确定性越强，模型自主权应越小。** 支付、退款、发货、权限变更等有副作用的流程，应由代码/工作流决定边界，让模型只承担分类、抽取、生成、检索与建议；高风险工具必须审批、幂等、可审计。
3. **短任务与长任务是两类架构。** 请求内完成的工具调用 Agent 可以使用轻量 SDK；跨分钟/小时、等待人工、允许断线或需要故障恢复的任务，必须选择带 checkpoint/resume 的运行时，或外接 Temporal、DBOS、Restate、Inngest、Workflow DevKit 等持久化执行系统。
4. **多 Agent 不是默认升级路径。** 只有当子任务具有不同权限、上下文、模型、生命周期或团队所有权时，才值得拆成多个 Agent。仅仅给同一个模型换几个“角色名”，通常只增加调用次数和调试难度。
5. **默认推荐不是单一框架，而是按栈选基线。**
   - Python 类型安全、普通工具 Agent：`Pydantic AI`。
   - Python/TypeScript 复杂有状态编排：`LangGraph`；接受更高的学习和建模成本。
   - TypeScript 全栈产品：简单 Agent 用 `Vercel AI SDK`，完整后端与工作流用 `Mastra`；需要持久化时分别评估 `WorkflowAgent` 或 Mastra 的 durable/Inngest 路线。
   - OpenAI-first、语音/实时、handoff 或 sandbox Agent：`OpenAI Agents SDK`。
   - GCP、多语言和 A2A：`Google ADK`；必须逐语言验证能力对齐。
   - Azure/.NET：`Microsoft Agent Framework`，新项目不再把 AutoGen 或 Semantic Kernel 当第一候选。
   - AWS/Bedrock-first 的轻量模型驱动 Agent：`Strands Agents`。
   - Java/Spring：`Spring AI` 作为模型、工具、MCP、RAG 接入层；复杂 Agent 编排另选 `Spring AI Alibaba Graph/Agent Framework`、`Google ADK Java/Kotlin` 或外部持久化工作流。`LangChain4j` 的 agentic 模块当前仍是实验性能力。[Spring AI][spring-ai] [Spring AI Alibaba][spring-ai-alibaba] [LangChain4j Agents][langchain4j-agents]
6. **框架选型必须由同一业务样例的故障注入 PoC 决定。** Hello World、天气工具和官方 demo 不能证明恢复、幂等、隔离、可观测、成本或升级能力。

## 1. 调研范围与术语

本文讨论代码优先的 AI 应用/Agent 开发框架，包括：

- 模型调用、结构化输出和 tool calling；
- 自主 Agent loop、handoff、subagent；
- 顺序、并行、路由、循环、图和事件驱动工作流；
- session、memory、checkpoint、resume 和 human-in-the-loop；
- MCP、A2A、AG-UI 等互操作协议；
- tracing、evaluation、部署与生产治理。

本文不把以下产品直接与代码框架混为一类：

- Dify、n8n、Flowise、Langflow 等低代码/可视化平台；
- 单纯的向量数据库、文档解析器或模型网关；
- 只提供模型推理 API、但不提供 Agent 运行时的模型 SDK；
- Temporal、DBOS、Restate、Inngest 等通用持久化执行系统。

这些产品可以成为整体方案的一层，但不能替代其他层的选型。

### 1.1 Workflow 与 Agent

本文采用如下边界：

```text
Workflow：路径主要由代码、图或事件规则预先定义，模型只在受控节点内工作。
Agent：模型在循环中根据上下文和工具结果，动态决定下一步、工具和结束条件。
```

二者不是非此即彼。生产系统更常见的形态是“确定性外壳 + 局部 Agent”：工作流控制预算、权限、审批和状态，Agent 只在一个有界节点内自主探索。

### 1.2 框架所在的层

```text
用户 / 渠道 / Agent UI
          |
API、鉴权、租户、限流、流式协议
          |
确定性工作流 / Agent loop / 多 Agent 编排
          |
Run state、checkpoint、queue、resume、scheduler
          |
模型适配器 ----- 工具网关 / MCP ----- 业务服务与数据
          |
模型供应商       权限与审批             权威业务状态

横切能力：OpenTelemetry、日志、评测、成本、安全策略、版本治理
```

框架可能覆盖其中一到数层。选型时必须问“能力具体落在哪一层、由谁持久化、故障后如何恢复”，不能只看功能清单里是否出现 `memory`、`workflow` 或 `production-ready`。

## 2. 先按复杂度分级

| 级别 | 典型形态 | 应采用的最小方案 | 不应提前引入 |
| --- | --- | --- | --- |
| L0 | 摘要、分类、抽取、结构化生成 | 模型 SDK + schema 校验 + eval | Agent、多 Agent、图运行时 |
| L1 | 3～10 次工具调用，请求内结束 | 轻量 Agent loop + 步数/成本上限 | 分布式 Agent、复杂 checkpoint |
| L2 | 固定分支、并行、重试、人工审批 | 显式 workflow/graph + 持久状态 | 让模型自由决定所有业务路径 |
| L3 | 跨进程、长时间、断点恢复、外部副作用 | durable runtime + 幂等工具 + checkpoint/resume | 仅靠内存 session 或 HTTP 长连接 |
| L4 | 跨团队/跨服务的 Agent 协作 | 稳定契约 + A2A/API + 独立身份与审计 | 同进程中为了“角色感”拆服务 |

多数业务应用停在 L1 或 L2。只有执行时间、故障恢复或组织边界真实存在时，才应进入 L3/L4。

## 3. 一票否决问题

在给框架打分前，先回答以下问题。任何“不满足”都应直接淘汰候选，而不是用总分补偿。

1. **主语言与运行环境是什么？** 是否允许新增 Python/Node 运行时？团队能否承担其发布、监控和安全基线？
2. **一次运行最长多久？** HTTP 请求内、几分钟、几小时，还是等待人工数天？
3. **进程重启后必须从哪里恢复？** 只恢复会话，还是恢复到具体工具调用/工作流节点？
4. **工具是否产生不可逆副作用？** 是否具备幂等键、重放保护、补偿、审批和审计？
5. **是否需要模型/云可替换？** “支持多个 provider”不等于推理、缓存、内置工具、实时语音和结构化输出完全可移植。
6. **数据能否进入厂商控制面？** trace、prompt、tool 参数和 memory 是否包含个人信息、密钥或商业数据？
7. **是否需要实时语音、浏览器 UI 或可恢复流？** 这是 Agent loop 之外的独立能力。
8. **部署必须在哪个云/网络区？** 是否要求私有化、离线模型、现有 IAM、KMS、审计平台和 OpenTelemetry？
9. **谁维护运行状态的 schema？** 框架升级后，旧 checkpoint、暂停审批和 session 是否还能恢复？
10. **验收标准是否可计算？** 如果无法定义任务成功、允许动作和成本上限，就无法可靠选型。

## 4. 主流框架现状

以下是能力定位，不是脱离业务的排行榜。“持久化”特指运行中断后可恢复，不只是保存聊天记录。

| 框架/生态 | 主要语言 | 编排模型 | 持久化与恢复 | 最适合 | 主要风险或代价 |
| --- | --- | --- | --- | --- | --- |
| OpenAI Agents SDK | Python、TypeScript | Agent loop、agent-as-tool、handoff | session + 可序列化 RunState；不是通用图引擎 | OpenAI-first、实时语音、sandbox、轻量多 Agent | OpenAI 高级能力最完整；跨 provider 仅能保证抽象层，不保证特性等价 |
| LangChain Agents / LangGraph | Python、TypeScript | 高层 Agent + 低层 state graph | checkpointer、store、interrupt、time travel | 长运行、有状态、路径需精细控制的复杂 Agent | 学习和状态建模成本高；完整观测/部署常会进入 LangSmith 产品栈 |
| Google ADK 2.0 | Python、Java/Kotlin、Go、TypeScript | Agent + graph/workflow + task delegation | session、event、resume；也有 durable runtime 集成 | GCP、A2A、多语言、多 Agent | 迭代快、2.0 有破坏性变化；不同语言的能力与成熟度不齐 |
| Microsoft Agent Framework | Python、.NET | Agent + graph workflow | checkpoint、restart、HITL、Durable hosting | Azure/Foundry、.NET、企业治理 | Azure 路线最顺；迁移期仍需验证包、宿主和文档稳定性 |
| Pydantic AI | Python | 类型化 Agent、graph、capability | 通过 Temporal/DBOS/Prefect/Restate 等官方集成实现 durable | Python 类型安全、FastAPI 风格、模型中立 | durable 不是单一内置运行时，组合方案由团队负责运维 |
| CrewAI | Python | role-based Crew + event-driven Flow | Flow state；企业控制面提供更多治理 | 快速搭建协作型自动化和业务演示 | 角色抽象易诱导过度多 Agent；企业治理能力与 AMP 商业产品关联较强 |
| LlamaIndex Workflows | Python 为主 | 类型事件驱动 workflow | checkpoint/resume 与 DBOS 路线 | RAG、文档、检索和数据密集型 Agent | 若核心不是数据/RAG，其抽象和集成优势会下降 |
| Mastra | TypeScript | Agent + schema workflow + memory | suspend/resume；durable agent 当前 beta，可接 Inngest | Node/TS 后端、需要完整 Agent 产品栈 | 能力面宽且变化快；durable beta 与平台功能要逐项验证 |
| Vercel AI SDK | TypeScript | ToolLoopAgent、代码工作流、WorkflowAgent | 普通 loop 在内存；WorkflowAgent 可持久恢复 | Web/Next.js、流式 UI、轻中度 Agent | 核心 SDK 与 Workflow DevKit/部署能力应分别评估，避免把前端便利等同于后端治理 |
| Strands Agents | Python、TypeScript | model-driven Agent loop、hooks、multi-agent patterns | 核心偏 loop；长期运行需结合部署/运行层 | AWS/Bedrock-first、轻量跨模型 Agent | 默认体验偏 AWS；复杂业务图与恢复需另行设计 |
| Claude Agent SDK | Python、TypeScript | Claude Code 同源 harness、内置文件/命令/搜索工具 | session、hooks、checkpoint 等 harness 能力 | 编码、研究、文件系统和长视野执行 | 面向 Claude 及其 harness；不适合作为普通 CRUD Agent 的中立编排层 |
| Spring AI | Java | 模型、工具、MCP、RAG、Advisor | chat memory；核心不等于 durable Agent graph | Spring Boot 中接入 AI 能力 | 需另配真正的 Agent 编排和 durable runtime |
| Spring AI Alibaba | Java | Agent Framework + Graph | Graph 提供状态、持久化、长运行编排 | Java/Spring、多 Agent、国内云与 Nacos 生态 | 相比 Python/TS 主流生态更年轻；升级和生产案例需自行验证 |
| LangChain4j | Java | AI Services、tool、RAG；agentic 模块 | 取决于应用/外部运行时 | 非 Spring 或多 JVM 框架的统一模型接入 | 官方仍把 agentic 模块标为 experimental，不宜直接承担高风险主流程 |

### 4.1 OpenAI Agents SDK

OpenAI Agents SDK 当前同时提供 Python 和 TypeScript，实现 agent、tool、guardrail、handoff/agent-as-tool、session、tracing、HITL、MCP、Realtime 和 sandbox agent。Python 官方仓库也明确支持 OpenAI API 以外的模型接入。[OpenAI Agents Python][openai-agents-python] [OpenAI Agents TypeScript][openai-agents-js]

值得肯定的点：

- 概念少，Agent loop 和多 Agent delegation 上手快；
- OpenAI Responses、hosted tools、Realtime、tracing 和 sandbox 的纵向集成顺畅；
- HITL 可以把 `RunState` 序列化后跨进程恢复，审批能覆盖 handoff 与嵌套 agent-as-tool。[OpenAI HITL][openai-agents-hitl]
- 适合把“一个 Agent + 少量专用 Agent/工具”快速推到产品。

边界与风险：

- 它不是 LangGraph 一类通用状态图引擎；复杂业务路由若全部编码在 handoff 和 callback 中，后期会难以审计；
- provider-agnostic 只表示存在适配入口，OpenAI 内置工具、实时能力、推理事件和 tracing 语义并不会自动移植；
- session、对话历史、可序列化暂停状态不能替代业务事务、队列和 durable workflow；
- sandbox Agent 能执行文件和命令，必须配置隔离、权限、网络和资源预算。

**推荐使用：** OpenAI-first、语音 Agent、客服 handoff、编码/研究 sandbox，以及中等复杂度的工具 Agent。

**不作为首选：** 大量确定性节点、跨天业务审批、强状态图审计，或要求所有模型特性完全中立的系统。

### 4.2 LangChain Agents / LangGraph

LangChain 当前把产品边界拆得比较清楚：LangChain Agents 是高层 Agent 框架，LangGraph 是低层编排运行时，LangSmith 承担 tracing、evaluation、deployment 等平台能力。LangGraph 本身不强制使用 LangChain 模型/工具抽象。[LangGraph Overview][langgraph-overview]

LangGraph 的核心优势是：

- 显式 state、node、edge，确定性步骤和模型驱动步骤可以混合；
- checkpointer 保存线程内图状态，store 保存跨线程长期信息；
- interrupt/HITL、故障恢复、time travel、subgraph 和流式事件是一等能力；
- 复杂路径可以被可视化、测试和回放，而不是隐藏在 prompt 里。[LangGraph Persistence][langgraph-persistence]

代价也很直接：

- 团队需要设计 state schema、reducer、节点边界、checkpoint 生命周期和兼容迁移；
- 图很容易被滥用成“每个函数一个节点”，形成比普通代码更难读的流程；
- 开源运行时、LangSmith 观测和托管部署是不同边界，采购与数据合规要分别判断；
- durable replay 不能替代副作用幂等。任何可能被重放的工具都要用业务幂等键保护。

**推荐使用：** 复杂路由、长运行、可恢复、人审节点、研究/分析 pipeline，以及状态转移本身就是核心业务的 Agent。

**不作为首选：** 只有一个模型和三五个工具、请求内结束的普通助手。

### 4.3 Google ADK 2.0

ADK 2.0 已把 Agent、graph workflow、task delegation、session、event、evaluation、observability、deployment、MCP、A2A 放入统一体系，并在官方文档中提供 Python、Java/Kotlin、Go、TypeScript 入口。[Google ADK][google-adk] [Google ADK Docs][google-adk-docs]

优势：

- workflow runtime 同时支持 routing、fan-out/fan-in、loop、retry、state、HITL 与嵌套流程；
- GCP 的模型、搜索、数据、Cloud Run/GKE/Agent Runtime 和观测集成完整；
- A2A 是一等能力，适合跨服务或跨团队 Agent；
- 代码优先，仍可脱离 Google Cloud 在本地或其他环境部署。

需要特别注意：

- Python ADK 2.0 官方说明包含 agent API、event model、session schema 的破坏性变化，发布节奏较快；
- “支持某语言”不代表所有语言 feature parity。当前 resume 文档明确标注支持 Python 与 Kotlin，并提示工具在恢复时是 **at least once**，可能重复执行。[Google ADK Resume][google-adk-resume]
- Java 官方仓库中的部分能力仍有 preview/coming soon 标记；必须用目标语言做完整 PoC，不能拿 Python demo 代替；
- 对非 GCP 团队，ADK 的最大优势会下降，运维面反而可能扩大。

**推荐使用：** GCP、Gemini/Vertex、需要 A2A、多 Agent workflow，或者希望在多语言组织内使用相近概念模型的团队。

### 4.4 Microsoft Agent Framework

Microsoft Agent Framework（MAF）是微软当前面向 Python 与 .NET 的统一 Agent/多 Agent workflow 框架，官方仓库把 graph workflow、checkpoint、streaming、HITL、time travel、OpenTelemetry、Foundry hosting 和 provider flexibility 作为核心能力，并提供从 AutoGen 与 Semantic Kernel 迁移的指南。[Microsoft Agent Framework][microsoft-agent-framework]

因此选型上应作如下处理：

- Azure/.NET 新项目优先评估 MAF；
- 已有 AutoGen/Semantic Kernel 项目先评估迁移成本，不必为了追新立即重写；
- 不建议新项目在 AutoGen、Semantic Kernel、MAF 三套抽象之间同时下注；
- Foundry 托管、Azure Durable hosting 与框架 OSS 核心的责任边界要分别验证。

**推荐使用：** .NET 团队、Azure/Foundry、企业身份治理和已有 Microsoft 技术栈。

### 4.5 Pydantic AI

Pydantic AI 的主要价值不是“更多 Agent 角色”，而是把 Python 类型、依赖注入、结构化输出、模型适配、工具 schema、测试、eval 和 OpenTelemetry 贯通。它支持 graph、MCP、HITL 与多 Agent pattern，也能把 durable execution 交给 Temporal、DBOS、Prefect、Restate 等官方集成。[Pydantic AI][pydantic-ai] [Pydantic Durable Execution][pydantic-durable]

优势：

- 输入、依赖、工具参数和输出的类型边界清楚，适合 FastAPI/Pydantic 团队；
- 模型中立程度较好，provider 扩展入口明确；
- eval 与 OTel 不只是外围宣传，而是框架设计中的一等能力；
- durable runtime 可以按团队已有基础设施选择。

代价：

- 组合式 durable 意味着部署、版本、重试语义和可观测整合仍由团队负责；
- 复杂 graph 的生态与可视化成熟度需要和 LangGraph 实测比较；
- 类型校验能阻止结构错误，不能证明内容正确、工具调用安全或业务决策合理。

**推荐使用：** Python/FastAPI、重视类型与测试、模型中立、已拥有 Temporal/DBOS/Restate 等运行时的团队。

### 4.6 TypeScript：Mastra 与 Vercel AI SDK

两者经常被同时比较，但定位不同。

`Vercel AI SDK` 首先是 provider-agnostic 的 TypeScript AI 应用工具箱，强项是模型调用、结构化输出、tool loop、流式响应和多前端框架 UI。`ToolLoopAgent` 默认有 20 步停止上限，适合请求内 Agent；需要持久恢复时，`@ai-sdk/workflow` 的 `WorkflowAgent` 把每次工具执行放进 Workflow DevKit 的 durable step，支持重启恢复、自动重试、可恢复审批与流。[Vercel AI SDK Agent][vercel-ai-agent] [Vercel WorkflowAgent][vercel-workflow-agent]

`Mastra` 更接近完整的 TypeScript Agent 后端框架，内置 agent、schema workflow、memory、RAG、MCP、observability、eval、server 和 deployment。普通 workflow 支持 suspend/resume；durable agent 当前官方标为 beta，生产执行可接 Inngest。[Mastra Workflows][mastra-workflows] [Mastra Durable Agent][mastra-durable]

建议：

- 主要问题是 Web chat、generative UI 和简单工具调用：先用 Vercel AI SDK；
- 需要独立 Agent 服务、workflow、memory、eval 和管理面：评估 Mastra；
- 已经使用 Mastra 后端，前端仍可使用 AI SDK UI 或其他 UI 协议；
- 不要因 Next.js 接入方便，就默认把长任务、审批和业务事务都放进请求处理器。

### 4.7 专项框架

#### CrewAI

CrewAI 用 `Crew` 表达自治角色协作，用 `Flow` 表达事件驱动的确定性流程与共享状态。[CrewAI][crewai]

它适合快速构建研究、内容、销售运营等“角色天然可理解”的自动化，也适合让非底层框架专家快速进入多 Agent 思维。但角色、backstory 和 delegation 很容易掩盖真正的控制流。高风险系统应把关键路径落到 Flow 和普通 Python 代码，而不是依赖自由协商。企业级 tracing、治理和控制面的实际需求还要评估 AMP 商业产品边界。

#### LlamaIndex Workflows

LlamaIndex Workflows 是类型化、事件驱动、async-first 的 step 模型：事件类型描述边，普通 Python 处理 branch、loop、并行、共享状态和资源。官方文档同时覆盖 HITL、durable workflow、DBOS、测试和 observability。[LlamaIndex Workflows][llamaindex-workflows]

它在 RAG、文档解析、检索、query planning、rerank 和数据 Agent 中最有优势。若系统的核心只是普通 SaaS 工具调用，LlamaIndex 的数据生态不一定带来足够收益。

#### Strands Agents

Strands 是 Python/TypeScript 的 model-driven Agent SDK，内置 hooks、guardrail、steering、MCP、streaming、structured output 和多 Agent pattern，默认体验围绕 Bedrock，但也支持 Anthropic、OpenAI、Gemini、Ollama 等 provider。[Strands Agents][strands-agents]

它适合 AWS/Bedrock-first 的轻量 Agent 与快速试验。复杂图、跨天恢复和强业务事务仍应配合明确的 workflow/durable 层。

#### Claude Agent SDK

Claude Agent SDK 把 Claude Code 的 Agent loop、上下文管理以及 Read/Edit/Bash/Glob/Grep/Web 等内置工具暴露给 Python 和 TypeScript，并提供 hooks、permissions、subagents、MCP、session、成本追踪与 OTel。[Claude Agent SDK][claude-agent-sdk]

这类 harness SDK 与普通 Agent 框架不同：它已经对文件系统、命令执行、搜索和长视野任务做了大量产品化。编码、研究、运维助手可以直接受益；普通客服或交易工作流若不需要这些能力，采用它会引入过大的权限面和 vendor/harness 绑定。

## 5. Java 技术栈的专项结论

Java 团队不应因为 Python Agent 生态更丰富，就自动把所有 AI 业务迁到 Python。先判断 AI 逻辑是否需要独立团队、独立扩缩容和独立发布；否则保留 Spring Boot 的业务边界通常更省成本。

### 5.1 推荐分层

```text
Spring MVC/WebFlux API
        |
认证、租户、限流、业务事务
        |
Agent Application Service
        |-------------------------------|
Spring AI：model/tool/MCP/RAG            |
                                        |
可选编排：Spring AI Alibaba Graph / ADK  |
可选 durable：Temporal/DBOS/其他工作流    |
        |
普通 Spring Domain Service + 数据库
```

原则：

- tool 只调用已有 application/domain service，不在 tool callback 中复制业务规则；
- Spring 事务不能跨 LLM 调用或人工等待长期持有；
- Agent run state 与业务权威状态分库存储或至少分 schema 管理；
- 通过幂等键把“模型重复调用”降级为可识别的重复请求；
- Spring AI 2.x 对应 Spring Boot 4.x，Spring AI 1.1.x 对应 Boot 3.5.x，不能脱离现有 Boot 基线选版本。[Spring AI][spring-ai]

### 5.2 Java 候选选择

| 场景 | 优先候选 | 说明 |
| --- | --- | --- |
| Spring Boot 中增加聊天、RAG、少量工具 | Spring AI | 与 Boot、观测、配置和数据生态一致 |
| Java 中构建复杂 graph/multi-agent | Spring AI Alibaba 或 ADK Java/Kotlin | 前者更贴近 Spring，后者更贴近 Google/A2A；均需真实 PoC |
| Quarkus/Micronaut/Helidon 或不想绑定 Spring | LangChain4j | 模型与向量库适配丰富，但 agentic 模块仍实验性 |
| 跨小时、等待人工、强恢复 | 上述框架 + durable runtime | 不让 Agent 框架独自承担业务事务正确性 |
| AI 团队独立于 Java 业务团队 | Python/TS Agent service + 明确 API/A2A | 只有组织和运行边界真实独立时才拆服务 |

## 6. 协议不是框架

### 6.1 MCP：工具接入协议

MCP 适合让多个 Agent 客户端复用同一工具/资源服务，或隔离不同语言与发布周期。它不提供业务工作流、事务、Agent memory 或任务恢复。

不建议“所有内部函数都 MCP 化”。同进程、单消费者、低延迟的普通函数工具更简单。只有在工具需要独立部署、多客户端复用、权限隔离或跨语言时，再承担协议、网络和鉴权成本。

MCP 也是新的安全边界。官方安全规范明确讨论 confused deputy、token passthrough、SSRF、redirect URI 和 OAuth state 等风险，并禁止把并非签发给 MCP server 的 token 直接向下游透传。[MCP Security][mcp-security]

### 6.2 A2A：远程 Agent 协作协议

A2A 适合跨服务、跨团队或跨组织发现与调用 Agent，并传递长任务状态。它不适合同一进程中本可用函数调用完成的 agent-as-tool。采用 A2A 的前提是每个 Agent 有独立身份、SLA、版本、权限和审计需求。[A2A][a2a]

### 6.3 AG-UI 与 UI 流协议

AG-UI 一类协议解决 Agent 到前端的事件流、工具状态、共享状态和人机交互。它不负责后端的 durable execution。前后端可以分别选型，通过稳定事件契约连接，而不必使用同一框架。[AG-UI][ag-ui]

## 7. 推荐的生产参考架构

### 7.1 状态必须分开

至少区分四类状态：

| 状态 | 例子 | 权威存储 | 生命周期 |
| --- | --- | --- | --- |
| 对话状态 | message、summary、附件引用 | session store | 会话级 |
| Agent run state | 当前节点、待审批工具、预算、checkpoint | run/checkpoint store | 单次运行 |
| 长期 memory | 用户偏好、已确认事实 | 专门 memory/store | 跨会话，可修订 |
| 业务状态 | 订单、退款、库存、权限 | 业务数据库 | 业务规则决定 |

向量库只是长期 memory 的一种检索手段，不是业务事实源。模型生成的 memory 在进入长期存储前应有来源、租户、时间、置信度和删除策略。

### 7.2 工具网关

所有产生副作用的工具建议经过统一网关：

```text
模型工具调用
   -> schema 校验
   -> tenant/user 身份绑定
   -> policy 与 allowlist
   -> 风险分级 / 人工审批
   -> 幂等键与并发控制
   -> domain service
   -> 审计事件与脱敏结果
```

工具设计要求：

- 一个工具表达一个清晰意图，参数名和边界对模型友好；
- 查询工具与写入工具分开，写入工具默认最小权限；
- 返回结构要小而稳定，大结果保存为引用并提供后续查询；
- 所有外部副作用支持幂等、超时、重试分类和可观测；
- 不把数据库连接、API key 或内部异常堆栈放入模型上下文；
- 对 shell、浏览器、文件系统和代码执行使用真正的 sandbox 与网络策略。

### 7.3 Durable execution 的语义

“可恢复”通常意味着步骤可能被重放，而不是 exactly-once。Google ADK 的 resume 文档也明确提示工具是至少执行一次，恢复时可能重复。[Google ADK Resume][google-adk-resume]

因此必须满足：

1. 每次 run、step、tool call 都有稳定 ID；
2. 写工具接收业务幂等键；
3. checkpoint 在副作用前后有明确边界；
4. 超时后把结果标记为“未知”而不是直接当失败重试；
5. 需要时使用 outbox、fencing token 或补偿动作；
6. 暂停任务同时保存 agent/prompt/tool schema 版本，避免新代码恢复旧状态。

### 7.4 可观测与评测

生产 trace 至少包含：

- `run_id`、`session_id`、tenant、用户与请求来源；
- model/provider/version、prompt 版本、temperature 等关键参数；
- 每步输入输出摘要、tool call、审批、重试、异常和结束原因；
- token、缓存命中、耗时、成本估算；
- checkpoint、resume、取消与重复执行；
- 最终业务结果与用户反馈。

优先让框架导出 OpenTelemetry，再接入现有平台。OpenTelemetry 已提供 GenAI agent、model 和 event 的语义约定，但语义仍会演进，内部字段应保留扩展空间。[OpenTelemetry GenAI][otel-genai]

评测不能只看最终文本。至少覆盖：

- task success / business outcome；
- 工具选择、参数正确率与非法动作率；
- 轨迹长度、无效循环、handoff/route 正确率；
- groundedness、引用质量和结构化输出有效率；
- 人工接管率与审批拒绝率；
- p50/p95 延迟、token、成本和超时率；
- 重启恢复、重复消息、工具超时和 provider 降级结果；
- 越权、prompt injection、跨租户与数据泄漏测试。

### 7.5 预算与结束条件

每次 run 必须同时设置：

- 最大模型调用次数；
- 最大工具调用次数和单工具次数；
- 最大墙钟时间；
- 最大 token/成本；
- 最大并发 subagent 数；
- 可重试错误清单和最大重试；
- 明确的 success、partial、failed、cancelled、expired 终态。

“模型自己判断已经完成”不能成为唯一结束条件。

## 8. 分场景推荐

| 场景 | 首选 | 次选 | 关键条件 |
| --- | --- | --- | --- |
| Python API，类型化工具与结构化输出 | Pydantic AI | OpenAI Agents SDK | 若流程复杂再引入 graph/durable 层 |
| 复杂研究、审批、状态图与故障恢复 | LangGraph | ADK 2.0 / MAF | 接受显式状态建模与更高工程成本 |
| OpenAI 实时语音、handoff、sandbox | OpenAI Agents SDK | 自建 provider SDK | 充分利用供应商能力，不虚构完全可移植性 |
| Claude 编码、文件、命令和研究 Agent | Claude Agent SDK | OpenAI Sandbox Agent | sandbox、权限、网络与成本必须单独治理 |
| Next.js/React 流式 Agent UI | Vercel AI SDK | Mastra + AI SDK UI | 长任务使用 WorkflowAgent 或独立 durable 后端 |
| TypeScript 完整 Agent 后端 | Mastra | Vercel AI SDK + Workflow DevKit | Mastra durable beta 要做恢复与升级 PoC |
| GCP/Vertex、多语言、A2A | Google ADK | LangGraph/Pydantic AI | 逐语言核对 feature parity |
| Azure/.NET/Foundry | Microsoft Agent Framework | 自建 Semantic Kernel 延续方案 | 新项目优先 MAF，旧项目按迁移收益决定 |
| AWS/Bedrock 轻量 Agent | Strands | Mastra/Pydantic AI | 复杂 workflow 与 durable 另配运行层 |
| RAG、文档与数据密集 Agent | LlamaIndex Workflows | Pydantic AI/LangGraph | 先验证 retrieval 质量，而非先扩成多 Agent |
| Java/Spring 简单 AI 功能 | Spring AI | LangChain4j | 保留现有业务服务与事务边界 |
| Java 复杂 Agent graph | Spring AI Alibaba / ADK Java/Kotlin | 独立 Python Agent service | 用目标版本、目标语言完成故障恢复 PoC |
| 受监管、高风险业务动作 | 显式 graph + durable runtime | 普通代码工作流嵌入局部 LLM | 默认不使用自由多 Agent 决策关键动作 |

## 9. 两阶段 PoC 与评分门禁

### 9.1 阶段一：一票否决验证

对每个候选用相同代码边界完成以下测试：

1. 对接两个目标模型，验证工具调用、结构化输出、流和错误语义；
2. 实现一个读工具、一个幂等写工具、一个需要人工审批的工具；
3. 运行到中途杀进程，重启并恢复；
4. 在工具完成但结果未写回时制造超时，验证不会重复产生副作用；
5. 模拟 provider 429/5xx、超时、无效 tool args 和上下文超限；
6. 验证 tenant、credential、trace、checkpoint 的隔离与脱敏；
7. 升级一个小版本，验证旧 session/checkpoint 是否可读；
8. 导出 OTel trace，确认能关联最终业务结果。

任何候选只要在必须能力上失败，就不进入打分阶段。

### 9.2 阶段二：加权评分

建议按项目调整权重，而不是复制固定总分：

| 维度 | 普通 SaaS Agent | 长运行 Agent | 说明 |
| --- | ---: | ---: | --- |
| 正确性与可控性 | 20 | 20 | 轨迹、结构化输出、非法动作 |
| 持久化与恢复 | 5 | 20 | checkpoint、resume、重放语义 |
| 工具与业务集成 | 15 | 10 | schema、DI、MCP、现有服务 |
| 安全与治理 | 15 | 15 | 身份、审批、隔离、审计 |
| 可观测与评测 | 15 | 10 | OTel、dataset、在线反馈 |
| 开发体验与可测试性 | 10 | 5 | 类型、调试、local dev、CI |
| 部署与运维 | 10 | 10 | 扩缩容、取消、队列、升级 |
| 生态与锁定成本 | 10 | 10 | provider、云、商业控制面 |
| **合计** | **100** | **100** | 先过硬门禁，再比较总分 |

### 9.3 数据集与统计

- 建立 50～200 条代表性离线 case，覆盖正常、边界、对抗和失败输入；
- 非确定性任务至少多次运行，报告均值、分位数和失败分布，不只展示最好结果；
- LLM-as-judge 只能补充，不应单独决定涉及业务事实或安全的通过/失败；
- 对每次框架、模型、prompt、tool schema 变更跑回归；
- 先选一个 champion 和一个 challenger，避免长期维护三套 PoC。

## 10. 降低锁定与迁移成本

建议把框架限制在 application/orchestration 层，并保留以下稳定边界：

```text
ModelGateway       # 只统一真正可统一的调用；供应商高级能力保留 escape hatch
ToolGateway        # 普通业务接口、JSON Schema、身份与幂等
RunRepository      # 独立的 run/checkpoint 领域模型
PolicyEngine       # 权限、审批、预算、数据分类
EventSink          # 统一运行事件与 OTel
EvaluationPort     # dataset、evaluator、结果版本
```

迁移友好的实践：

- prompt、tool schema、eval dataset 和 policy 独立版本化；
- 工具实现是普通可测试函数/服务，框架 decorator 只做薄适配；
- 保存必要的原始 provider event，同时输出内部规范化事件；
- 不让框架的 message/session 对象直接成为业务数据库模型；
- 不把核心业务规则写进 prompt、callback 或某个托管控制面；
- MCP/A2A 只用于真实跨边界，不把协议本身当成架构目标；
- 每季度用 challenger 重放核心 dataset，监控替换成本和能力漂移。

## 11. 常见误区

1. **按 GitHub star 选框架。** star 反映关注度，不证明恢复语义、兼容性和生产治理。
2. **把 memory 当数据库。** memory 是模型上下文策略；业务事实必须回到权威存储。
3. **宣称 provider-agnostic 就能无损换模型。** tool calling、推理、缓存、内置搜索、实时音频和内容过滤都存在语义差异。
4. **用更多 Agent 提高准确率。** 没有独立上下文、权限和评测时，多 Agent 往往只是更多随机调用。
5. **用自动重试解决可靠性。** 写操作自动重试会制造重复副作用，必须先设计幂等。
6. **把 tracing 等同于 evaluation。** trace 解释发生了什么，eval 判断结果是否达到业务标准。
7. **把 chat history 等同于 durable state。** 保存消息无法恢复到具体工具或工作流步骤。
8. **把 MCP 当安全层。** MCP 扩大工具复用的同时也新增 OAuth、SSRF、token 和供应链风险。
9. **在第一版就构建通用 Agent 平台。** 先用一个真实场景验证可复用边界，再平台化。
10. **直接照搬官方 demo 的部署方式。** demo 通常省略租户、秘密管理、取消、并发、数据保留和故障恢复。

## 12. 最终建议

对没有额外上下文的新项目，采用以下默认决策顺序：

```text
1. 单次调用/RAG 能否满足？
   能 -> 模型 SDK + schema + eval，结束。
   不能 -> 进入 2。

2. 路径是否可预先定义？
   能 -> 普通代码 workflow；需要恢复则加入 durable runtime。
   不能 -> 进入 3。

3. 是否只需请求内的有限工具 loop？
   是 -> 按语言选择 Pydantic AI / OpenAI Agents / Vercel AI SDK / Strands。
   否 -> 进入 4。

4. 是否需要显式状态、人工中断和恢复？
   是 -> LangGraph / ADK / MAF / Mastra+durable / LlamaIndex Workflows。
   否 -> 保持轻量 Agent，不引入 graph。

5. 是否存在跨服务、跨团队的 Agent 所有权？
   是 -> 再评估 A2A。
   否 -> 使用函数、tool 或 agent-as-tool，不拆分布式多 Agent。
```

如果必须给出一个通用起点：

- **Python 新项目：** 简单/中等复杂度先用 Pydantic AI；明确需要复杂长运行图时直接用 LangGraph。
- **TypeScript 新项目：** UI 和轻 Agent 用 Vercel AI SDK；完整 Agent 后端用 Mastra；长任务从第一天验证 durable 路线。
- **Java 新项目：** 先用 Spring AI 接入模型与工具，复杂编排再评估 Spring AI Alibaba/ADK，不把实验性 agentic 模块直接放到高风险主链路。
- **.NET/Azure：** Microsoft Agent Framework。
- **GCP：** Google ADK。
- **AWS/Bedrock：** Strands 作为轻量入口，复杂运行另配 workflow/durable 层。
- **OpenAI/Claude 专属高级能力：** 接受合理厂商绑定，直接使用其 Agents/Agent SDK；不要为了形式上的中立放弃关键能力，又在内部重造一套不完整 harness。

真正决定成败的通常不是 `Agent` 类的 API，而是工具边界、运行状态、幂等恢复、评测数据、安全策略和团队能否理解每一层发生了什么。

## 参考资料

以下资料均为官方一手来源，访问日期均为 2026-07-22。

[anthropic-effective-agents]: https://www.anthropic.com/engineering/building-effective-agents "Anthropic - Building effective agents"
[openai-agents-python]: https://github.com/openai/openai-agents-python "OpenAI Agents SDK for Python"
[openai-agents-js]: https://github.com/openai/openai-agents-js "OpenAI Agents SDK for JavaScript/TypeScript"
[openai-agents-hitl]: https://openai.github.io/openai-agents-python/human_in_the_loop/ "OpenAI Agents SDK - Human in the loop"
[langgraph-overview]: https://docs.langchain.com/oss/python/langgraph/overview "LangGraph overview"
[langgraph-persistence]: https://docs.langchain.com/oss/python/langgraph/persistence "LangGraph persistence"
[google-adk]: https://github.com/google/adk-python "Google Agent Development Kit"
[google-adk-docs]: https://adk.dev/ "Google ADK documentation"
[google-adk-resume]: https://adk.dev/runtime/resume/ "Google ADK - Resume stopped agents"
[microsoft-agent-framework]: https://github.com/microsoft/agent-framework "Microsoft Agent Framework"
[pydantic-ai]: https://pydantic.dev/docs/ai/ "Pydantic AI"
[pydantic-durable]: https://pydantic.dev/docs/ai/capabilities/durable_execution/overview/ "Pydantic AI durable execution"
[crewai]: https://github.com/crewAIInc/crewAI "CrewAI"
[llamaindex-workflows]: https://developers.llamaindex.ai/python/llamaagents/workflows/ "LlamaIndex Agent Workflows"
[mastra-workflows]: https://mastra.ai/docs/workflows/overview "Mastra workflows"
[mastra-durable]: https://mastra.ai/docs/long-running-agents/durable-agents "Mastra durable agents"
[vercel-ai-agent]: https://ai-sdk.dev/docs/agents/overview "Vercel AI SDK Agents"
[vercel-workflow-agent]: https://ai-sdk.dev/docs/agents/workflow-agent "Vercel AI SDK WorkflowAgent"
[strands-agents]: https://github.com/strands-agents/harness-sdk "Strands Agents"
[claude-agent-sdk]: https://code.claude.com/docs/en/agent-sdk/overview "Claude Agent SDK"
[spring-ai]: https://github.com/spring-projects/spring-ai "Spring AI"
[spring-ai-alibaba]: https://github.com/alibaba/spring-ai-alibaba "Spring AI Alibaba"
[langchain4j-agents]: https://docs.langchain4j.dev/tutorials/agents "LangChain4j Agents and Agentic AI"
[mcp-security]: https://modelcontextprotocol.io/specification/latest/basic/security_best_practices "MCP Security Best Practices"
[a2a]: https://github.com/a2aproject/A2A "Agent2Agent Protocol"
[ag-ui]: https://github.com/ag-ui-protocol/ag-ui "AG-UI Protocol"
[otel-genai]: https://opentelemetry.io/docs/specs/semconv/gen-ai/ "OpenTelemetry GenAI Semantic Conventions"
