---
title: Agent 框架分层与选型 QA 总结
date: 2026-07-24
tags:
  - AI-Agent
  - framework-selection
  - architecture
aliases:
  - Agent 框架选型问答
related: "[[ai-agent-framework-selection-research]]"
---

# Agent 框架分层与选型 QA 总结

> [!abstract]
> 本文总结围绕 [[ai-agent-framework-selection-research|AI Agent 开发框架技术选型调研（2026 全景）]] 的一组问答，重点澄清 Agent 循环、L1/L2 分层，以及 LangChain、LangGraph、Spring AI 和 LangChain4j 之间的关系。

## 一、核心结论

1. **Agent 循环是一种控制流模式，不是一组指定框架。** 只要模型能根据上下文和工具结果动态决定下一步，就已经具备 Agent 循环行为。
2. **框架分层看的是主要职责，不是代码中有没有循环。** 一个 L1 框架也能提供简单 Agent 循环，但不一定具备完整的持久化运行时能力。
3. **框架可以跨层。** 高层 API、底层运行时和具体执行行为可能分别处于不同层次。
4. **先用满足需求的最小方案。** 单次生成不必使用 Agent，请求内的简单工具循环不必直接设计状态图，只有出现长运行、人工审批和故障恢复需求时才需要强化运行时。

## 二、Agent 循环必须使用专门框架吗

不必须。OpenAI Agents SDK、Claude Agent SDK、Pydantic AI、Mastra 和 Vercel AI SDK 是以 Agent 循环为主要抽象的代表，但 Agent 循环本身可以由普通代码实现：

```text
用户输入
  -> 调用模型
  -> 模型决定是否调用工具
  -> 执行工具并回填结果
  -> 再次调用模型
  -> 直到完成或触发步数、时间、成本上限
```

可选实现包括：

- 使用 LangChain Agent；
- 使用其他 Agent SDK；
- 直接使用模型 SDK 和 Tool Calling 自己编写循环。

框架的价值不在于发明了这个循环，而在于替团队处理消息、工具协议、状态、恢复、审批、追踪和评测等工程问题。

## 三、什么是“请求内的简单工具循环”

“请求内”表示整个 Agent 运行在一次业务调用结束前完成。一次外部调用中可以包含多次模型和工具调用：

```text
用户：查北京天气并给出穿衣建议
  -> LLM 判断需要天气数据
  -> 调用 get_weather("北京")
  -> 工具返回 12°C、有风
  -> LLM 根据工具结果生成建议
  -> 返回最终答案
```

外部代码可能只有一次调用：

```python
result = agent.invoke({
    "messages": [
        {"role": "user", "content": "查北京天气并给出穿衣建议"}
    ]
})
```

但 `invoke()` 内部执行的是：

```text
LLM -> Tool -> LLM -> Tool -> LLM -> 最终答案
```

这类运行通常具备以下特征：

- 几秒到几分钟内完成；
- 只有少量工具调用；
- 执行期间应用进程持续存活；
- 中间状态主要位于内存；
- 不等待数小时或数天的人工审批；
- 不要求进程重启后从中间步骤继续。

如果任务需要长时间等待、重启恢复、审批后继续或避免关键副作用被重复执行，它就不再只是请求内循环，而是长运行、可恢复的工作流。

## 四、LangChain Agent 属于哪一层

需要区分框架定位、API 和运行行为：

| 观察对象 | 层次与定位 |
| --- | --- |
| LangChain 整体 | 以 L1 应用/RAG 编排为主 |
| LangChain Agent 高层 API | L1 提供的易用门面 |
| `LLM -> Tool -> LLM` 的执行行为 | L2 类型的 Agent 循环 |
| 现代 `create_agent()` 的底层运行时 | 通常是 L2 的 LangGraph |

因此，更准确的表述是：

> [!important]
> LangChain 是以 L1 为主的应用框架；LangChain Agent 是其高层 Agent 接口，执行的是 L2 循环，现代实现通常由 LangGraph 承担底层运行时职责。

一个简单的 `while` 循环也可以表现出 L2 行为，但不等于拥有完整的 L2 生产运行时。完整运行时还需要回答：

- 如何 checkpoint；
- 如何在进程重启后恢复；
- 如何暂停等待人工审批；
- 如何回放和迁移旧状态；
- 如何管理并行分支；
- 如何防止工具副作用在重放时重复发生。

## 五、LangChain Agent 是否依赖 LangGraph

需要区分“Agent 循环”与“现代 LangChain Agent 的标准实现”：

- **现代 LangChain `create_agent()`**：底层依赖 LangGraph，但使用者不必直接编写节点、边和状态图；
- **Agent 循环这一模式**：不依赖 LangGraph，可以自己实现或采用其他 Agent SDK；
- **旧式 `AgentExecutor`**：不以 LangGraph 为底层，但属于较早、较轻的实现，不是现代 LangChain 新项目的默认方向。

```text
业务代码
  -> LangChain create_agent()  # 高层 API
  -> LangGraph runtime         # 循环、状态与执行
  -> Model / Tool
```

## 六、为什么 Spring AI 被归为企业内核

Spring AI 核心并不是 Java 版 LangGraph，它更接近 Java/Spring 生态的 LangChain。**Spring AI Alibaba Graph** 才更接近 Java 生态的 LangGraph。

| 维度 | LangGraph | Spring AI 核心 | Spring AI Alibaba Graph |
| --- | --- | --- | --- |
| 核心抽象 | State、Node、Edge、Graph | ChatModel、ChatClient、Advisor、Tool、VectorStore | Graph、Sequential、Parallel、Routing、LoopAgent |
| 主要职责 | 有状态 Agent 执行 | 把 AI 能力接入 Spring 应用 | Java Agent 图编排 |
| 显式控制流 | 原生核心 | 主要由 Java/Spring 业务代码控制 | 原生支持 |
| checkpoint/resume | 核心能力 | 不是核心定位 | 提供图运行时能力，具体语义需 PoC |
| 工程取向 | Agent 运行时 | Spring DI、配置、治理与业务集成 | Spring 生态中的 Agent 运行时 |

Spring AI 被归入企业内核，是因为它首先回答：

> 如何把模型、RAG、工具和观测能力可靠地装入现有 Spring 企业应用？

LangGraph 首先回答的则是：

> 如何把 Agent 表达为一个可持久化、可中断、可恢复、可回放的状态图？

文档中的四种控制流哲学并不完全正交：“确定性图、Agent 循环、多智能体”描述控制流，“企业内核”更多描述工程集成取向。因此 Spring AI 可以跨类：

```text
Spring AI 核心
    -> 企业内核取向

Spring AI + Alibaba GraphAgent
    -> 企业内核 + 确定性图编排

Spring AI + 自定义工具循环
    -> 企业内核 + Agent 循环
```

## 七、LangChain4j 与 LangGraph 的区别

LangChain4j 更接近 Java 版 LangChain，而不是 Java 版 LangGraph。

| 维度 | LangChain4j | LangGraph |
| --- | --- | --- |
| 主要语言 | Java | Python、TypeScript |
| 主要层次 | L1 应用/RAG 编排 | L2 Agent 运行时 |
| 核心抽象 | `AiServices`、Model、Tool、Memory、RAG | State、Node、Edge、Graph、Checkpointer |
| Agent 循环 | 支持，但 agentic 模块仍偏实验性 | 核心能力 |
| 显式状态图 | 不是主要抽象 | 核心抽象 |
| checkpoint/resume | 不是核心强项 | 原生重点 |
| 适用方向 | Java 中快速接入模型、工具和 RAG | 复杂、有状态、可恢复的 Agent 工作流 |

二者没有必然依赖关系。Java 项目可以按需求选择：

```text
聊天、RAG、少量工具调用
    -> LangChain4j

Spring Boot 项目
    -> 通常优先 Spring AI

复杂状态图、路由、并行、循环
    -> Spring AI Alibaba Graph 或 Google ADK Java

跨小时、等待审批、强恢复
    -> Agent 框架 + durable runtime

必须使用 LangGraph 的完整能力
    -> 独立 Python/TypeScript LangGraph 服务
```

## 八、什么是“模型 SDK + schema + eval”

这是不引入 Agent 的最小 AI 应用方案：

```text
业务输入
  -> 使用模型 SDK 直接调用模型
  -> 模型按照固定 schema 输出
  -> 程序校验结果
  -> 返回业务系统

同时使用 eval 数据集持续验证质量
```

### 1. 模型 SDK

负责调用模型，例如 OpenAI、Anthropic、Google、通义、DeepSeek 或豆包 SDK，也可以使用 Spring AI `ChatModel`、`ChatClient` 或 LangChain4j `ChatModel`。

此时通常只有一次或固定次数的模型调用，没有模型自主决定下一步的工具循环。

### 2. Schema

Schema 是模型输出的数据契约，不是数据库表结构。例如将自由文本约束为：

```json
{
  "intent": "DELIVERY_DELAY",
  "orderId": "123",
  "urgency": 4,
  "summary": "用户催促订单发货"
}
```

程序随后验证枚举、字段格式、数值范围和必填项。Schema 能保证输出可被程序消费，但不能证明内容一定正确。

### 3. Eval

Eval 是一套可重复运行的模型效果测试。它使用代表性数据集统计：

- 分类和字段提取准确率；
- schema 合法率；
- 非法动作与拒绝率；
- 延迟、token 和成本；
- 边界、对抗和失败场景表现。

每次修改模型、Prompt 或 Schema 后都应重新运行。涉及业务事实或安全时，不能只依赖 LLM-as-judge。

该方案适合分类、抽取、摘要、翻译、审核和结构化生成。只有当模型需要根据中间结果动态选择工具及下一步时，才需要引入 Agent 循环。

## 九、快速判断表

| 需求 | 最小建议 |
| --- | --- |
| 分类、抽取、摘要、结构化生成 | 模型 SDK + schema + eval |
| RAG、固定步骤处理 | LangChain、Spring AI 或 LangChain4j |
| 请求内少量动态工具调用 | LangChain Agent、轻量 Agent SDK 或自定义循环 |
| 显式分支、并行、HITL、checkpoint | LangGraph、Spring AI Alibaba Graph、ADK 等 |
| 跨小时、等待审批、关键业务副作用 | Agent 框架 + durable runtime + 幂等工具 |

最终选型不要只问“能不能运行 Agent 循环”，而要继续问：

1. 运行是否会跨进程或长时间等待；
2. 是否要求从具体步骤恢复；
3. 工具是否会产生不可逆副作用；
4. 是否需要人工审批、回放和审计；
5. 团队能否理解并维护对应层次的状态与执行语义。
