# 第11章 团队协作——子Agent编排

> "一个人的精力有限，任务太大时，实习生知道请同事帮忙分工。但问题是：怎么分工？谁做什么？做完了怎么汇总？子Agent编排，就是让Agent学会这套'团队协作'。"

---

## 11.1 开篇故事：小李请同事帮忙

实习生小李接到了一个大任务："整理公司过去一年所有产品的用户反馈，按产品分类、按问题类型统计，下周给我一份完整报告。"

小李估算了一下——他一个人干不完。他决定请两个同事帮忙。

**第一层：怎么分工**

小李没有把整份资料随便丢给同事。他做了明确的拆分："小王，你负责产品A和产品B的反馈；小张，你负责产品C和产品D的反馈。每个人只需要整理自己负责的部分，最后汇总给我。"

**第二层：各干各的，互不干扰**

小王和小张各自在自己的工位上工作。小王读到的文件不会影响小张，小张写错的笔记也不会跑到小王的笔记本上。他们各自独立完成自己的部分。

**第三层：做完了怎么汇总**

两天后，小王和小张分别把整理好的表格交给小李。小李没有直接把两份表格发给主管——他先做了一遍汇总：把四个产品的数据合并成一张总表，检查有没有重复统计，最后写了一段总结性的分析结论。

**第四层：防止无限分包**

小李有个底线——他不会再把小王的工作转包给小赵。如果每个人都继续往下分包，最后全公司都在"整理用户反馈"，没人干正事了。

**Agent的子Agent编排，就是这位实习生的"团队协作术"。**

但Agent面临一个实习生没有的问题：LLM的上下文窗口有限，如果一个任务需要读50个文件、做10次搜索，所有中间结果都会挤占父Agent的上下文空间。子Agent编排要做的，就是把"大任务拆成小任务"、让子任务在独立的上下文中执行、最后把结果汇总回来——同时防止无限递归分包。

---

## 11.2 业务概念：子Agent编排是什么

**子Agent编排（Sub-Agent Orchestration）**是Agent系统的"任务委派系统"。它负责将复杂任务分解为可独立执行的子任务，派生具备独立上下文的子Agent执行，并将结果回传给父Agent汇总。

### 技术定义

子Agent编排不是简单的"多开几个线程"。它包含三个核心问题：

1. **怎么拆？**——任务分解策略：按文件拆分、按功能拆分、按时间拆分
2. **怎么跑？**——执行隔离：子Agent有独立的上下文、工具集、预算
3. **怎么回？**——结果回传：子Agent完成后，如何把结果交付给父Agent

> 💡 **名词解释：上下文隔离（Context Isolation）**
>
> 通俗地说：小李让同事帮忙时，给了每人一个独立的笔记本——同事记的笔记不会自动出现在小李的笔记本上，除非同事主动把结果交给他。
>
> 技术定义：子Agent拥有独立的`messages`数组、独立的会话标识、独立的工具集。子Agent的工具调用结果不会自动回灌到父Agent的上下文中，防止"大任务污染小会话"。

### 一句话说清边界

- **子Agent编排负责**：任务拆分、子Agent生命周期管理、结果汇总、深度限制
- **子Agent编排不负责**：具体工具怎么实现（那是工具系统）、子Agent的安全审批（那是安全防护）、父子会话的持久化存储（那是记忆系统）

### 没有子Agent编排会怎样？

如果把子Agent编排从Agent中拿掉：

- **上下文爆炸**：读50个文件的任务会让父Agent的上下文一次性增加上万token，触发强制压缩，丢失早期决策
- **长任务阻塞交互**：一个耗时30分钟的任务占用主Agent的全部注意力，期间用户无法发送新消息
- **子任务失败影响全局**：某个子任务（如"分析第3个模块"）失败，没有隔离机制会导致整个会话失败
- **资源无限递归**：Agent可能递归生成子Agent（A生成B，B生成C），最终耗尽系统资源

---

## 11.3 必要设计：所有Agent都必须有的"底线"

从四个项目中提炼共性，子Agent编排的**最小可行实现**只需要以下四个要素：

### 要素一：任务分解接口

父Agent必须有一个明确的方式"请求"子Agent——无论是通过工具调用（`delegate_task`/`spawn`）还是通过特殊消息。

### 要素二：上下文隔离

子Agent必须有自己的`messages`数组，与父Agent隔离。子Agent的工具调用、中间推理不应该污染父Agent的上下文。

### 要素三：结果回传

子Agent完成后，必须有一种机制把结果交付给父Agent。最简单的方式：子Agent完成后生成一段摘要，通过消息注入父会话。

### 要素四：深度限制

必须限制嵌套深度（如最多2层），防止Agent无限递归生成子Agent。

```python
# 所有Agent都必须有的"最小可行子Agent编排"
# 就像实习生的"最低配置团队协作"

class MinimalSubagentSystem:
    MAX_DEPTH = 2  # 最多2层嵌套
    
    async def delegate(self, parent, task_description):
        # ① 深度检查
        if getattr(parent, "_subagent_depth", 0) >= self.MAX_DEPTH:
            return {"error": "Max subagent depth reached"}
        
        # ② 创建子Agent（独立上下文）
        child = SubAgent(
            system_prompt=build_focused_prompt(task_description),
            tools=filter_tools(parent.tools),  # 子集，去掉敏感工具
            depth=parent._subagent_depth + 1,
        )
        
        # ③ 子Agent独立运行
        result = await child.run(task_description)
        
        # ④ 结果摘要回传父Agent
        summary = f"[Subagent completed]\n{result.summary}"
        parent.inject_message(role="system", content=summary)
        
        return {"summary": result.summary}
```

**这就是"底线"。**少于这四点，Agent的子Agent编排就不完整：

| 要素 | 少了会怎样 |
|------|-----------|
| 任务分解接口 | 父Agent无法触发子任务，所有任务挤在单一会话 |
| 上下文隔离 | 子任务中间结果污染父上下文，token爆炸 |
| 结果回传 | 子Agent做完了，父Agent永远收不到结果 |
| 深度限制 | Agent无限递归分包，最终耗尽资源 |

---

## 11.4 四个项目的实现对比

四个框架在"底线"之上，基于各自的执行模型（同步vs异步、阻塞vs后台）做了截然不同的扩展。

### Hermes Agent："完整实例+同步阻塞"的并发委托

**设计定位**：子Agent是完整的AIAgent实例、同步阻塞执行、支持并发——适合需要"并行外包"复杂子任务的高风险网关型Agent。

> 💡 **名词解释：Delegate Task（委托任务）**
>
> 通俗地说：小李把"整理竞品A的定价"这个任务写成一张便签，交给同事小王。小李站在原地等小王做完——不是因为他没事干，而是因为这份报告要按顺序写，他必须先拿到小王的结果才能继续。
>
> 技术定义：Hermes的`delegate_task`工具让父Agent把目标+受限工具集打包成一个临时的子AIAgent实例。子Agent独立运行完整的`run_conversation()`循环，父Agent阻塞等待所有子结果返回。支持批量并发（ThreadPoolExecutor）和级联中断。

> 💡 **名词解释：Leaf vs Orchestrator（叶子与 orchestrator 角色）**
>
> 通俗地说：公司有两种临时工——"干活的"（leaf）和"能再雇人的"（orchestrator）。小李请来的小王如果是"干活的"，他只能自己完成任务；如果是"能再雇人的"，他还可以再请小赵帮忙。但公司规定：最多只能雇两层。
>
> 技术定义：Hermes对子Agent进行角色分层。`leaf`角色只能执行工具，不能再委托；`orchestrator`角色保留`delegation`工具集，可以继续生成子Agent。是否保留orchestrator由运行时深度计数+全局开关决定，不是LLM说了算。

**关键源码位置**：
- 委托任务入口：`tools/delegate_tool.py:1870`
- 子Agent实例化：`tools/delegate_tool.py:1086-1116`
- 深度限制：`tools/delegate_tool.py:1912-1924`
- 角色降级：`tools/delegate_tool.py:900-908`
- 并发执行与中断：`tools/delegate_tool.py:2064-2107`
- 父子文件状态提醒：`tools/delegate_tool.py:1462-1733`
- 子Agent注册表：`tools/delegate_tool.py:183-216`

```python
# Hermes的子Agent编排：像一家有项目管理的大型外包公司

class HermesDelegationSystem:
    def delegate_task(self, parent, tasks):
        # ① 准入控制：深度检查 + 全局暂停开关
        depth = getattr(parent, "_delegate_depth", 0)
        if depth >= self.max_spawn_depth:
            return {"error": "Max depth reached"}
        
        # ② 角色降级：只有深度允许时才保留 orchestrator 能力
        orchestrator_ok = self.orchestrator_enabled and (depth + 1 < self.max_spawn_depth)
        effective_role = "orchestrator" if orchestrator_ok else "leaf"
        
        children = []
        for task in tasks:
            # ③ 构建子Agent：完整AIAgent实例，但精简配置
            child = AIAgent(
                ephemeral_system_prompt=build_focused_prompt(task.goal),
                enabled_toolsets=intersect(parent.toolsets, task.toolsets) - BLOCKED_TOOLS,
                quiet_mode=True,           # 子Agent不直接向用户输出
                skip_memory=True,          # 不写入父Agent的记忆
                iteration_budget=None,     # 全新预算
            )
            child._delegate_depth = depth + 1
            child._subagent_id = f"sa-{uuid}"
            parent._active_children.append(child)  # 用于级联中断
            children.append(child)
        
        # ④ ThreadPoolExecutor并发执行，每0.5秒检查父Agent是否被中断
        with ThreadPoolExecutor(max_workers=self.max_concurrent) as pool:
            futures = [pool.submit(run_child, c) for c in children]
            pending = set(futures)
            while pending:
                if parent._interrupt_requested:  # 用户喊停
                    for child in pending:
                        child.interrupt()  # 级联中断子Agent
                    break
                done, pending = wait(pending, timeout=0.5)
                for f in done:
                    results.append(f.result())  # 含summary、cost、status
        
        # ⑤ 父子文件交叉提醒：子Agent改了父Agent读过的文件？主动提醒
        for result in results:
            modified = check_sibling_writes(parent, result.task_id)
            if modified:
                result.summary += f"\n[NOTE: subagent modified files: {modified}]"
        
        return {"results": results}
```

### nanobot："后台异步+消息注入"的沙箱线程

**设计定位**：子Agent是在主循环内部开辟的独立沙箱推理线程——后台运行、不阻塞用户、完成后以system消息注入——适合个人助手场景。

> 💡 **名词解释：Spawn Tool（生成工具）**
>
> 通俗地说：小李对助手说"帮我后台查一下这个资料，查完了告诉我"——然后他继续处理手头的事，助手在后台默默工作，做完后轻声提醒他。
>
> 技术定义：nanobot的`SpawnTool`让LLM在推理过程中"决定"是否生成子Agent。子Agent以独立的`asyncio.Task`在后台运行，父Agent立即返回响应（"任务已启动，完成后通知你"），不阻塞用户交互。

> 💡 **名词解释：System Message Injection（系统消息注入）**
>
> 通俗地说：同事小王做完工作后，不是直接给用户打电话汇报，而是把便签贴在小李的办公桌上——小李看到后自己决定怎么跟用户说。
>
> 技术定义：nanobot的子Agent完成后，通过`MessageBus`以`channel="system"`的消息注入主会话。`system`消息在`loop.py`中被特殊处理：不走`_save_turn()`（不污染用户历史），直接触发主Agent生成自然语言总结。

**关键源码位置**：
- Spawn工具：`nanobot/agent/tools/spawn.py:11-63`
- 子Agent管理器：`nanobot/agent/subagent.py:50-77`
- 独立ToolRegistry：`nanobot/agent/subagent.py:92-108`
- 15轮循环：`nanobot/agent/subagent.py:121-155`
- 结果注入：`nanobot/agent/subagent.py:168-197`
- System消息路由：`nanobot/agent/loop.py:364-382`

```python
# nanobot的子Agent编排：像一个有后台助手的个人工作室

class NanobotSubagentSystem:
    async def spawn(self, task, label, origin):
        task_id = generate_task_id()
        
        # ① 创建后台asyncio.Task，不阻塞父Agent
        bg_task = asyncio.create_task(self._run_subagent(task_id, task, label, origin))
        
        # ② 注册双向索引：task_id→Task + session_key→{task_ids}
        self._running_tasks[task_id] = bg_task
        self._session_tasks[origin.session_key].add(task_id)
        bg_task.add_done_callback(self._cleanup)  # 自动清理
        
        # ③ 立即返回，告诉用户"任务已启动"
        return f"Subagent [{label}] started. I'll notify you when it completes."
    
    async def _run_subagent(self, task_id, task, label, origin):
        # ④ 独立ToolRegistry：去掉message和spawn，防止递归和骚扰用户
        tools = ToolRegistry()
        tools.register(ReadFileTool(...))
        tools.register(WriteFileTool(...))
        # 注意：没有tools.register(MessageTool)
        # 注意：没有tools.register(SpawnTool)
        
        # ⑤ 精简system prompt + 15轮ReAct循环（主循环40轮）
        messages = build_subagent_prompt(task)
        for i in range(15):
            response = await llm.chat(messages, tools.get_definitions())
            # ... 执行工具、回灌结果 ...
        
        # ⑥ 通过Bus以system消息注入主会话
        announce = f"[Subagent '{label}' completed]\nTask: {task}\nResult: {result}\nSummarize for user."
        await self.bus.publish_inbound(InboundMessage(
            channel="system",           # system通道=特殊路由
            sender_id="subagent",
            chat_id=f"{origin.channel}:{origin.chat_id}",
            content=announce,
        ))
    
    async def cancel_by_session(self, session_key):
        # 按会话精确取消：找到该会话的所有后台子Agent
        tasks = [self._running_tasks[tid] for tid in self._session_tasks.get(session_key, [])]
        for t in tasks:
            t.cancel()
```

### OpenClaw："事件驱动+深度守卫"的异步委派

**设计定位**：子Agent是独立的会话、事件驱动的结果交付、严格的深度限制——适合7×24小时运行的生产级网关。

> 💡 **名词解释：Announce（结果宣告）**
>
> 通俗地说：小王做完工作后，不是等小李来问，而是主动走到小李工位前说"我做完了，这是结果"。如果小李不在座位上，小王会在15分钟后再来一次——最多来3次。
>
> 技术定义：OpenClaw的`subagent-announce.ts`采用push模式：子Agent完成时主动将结果推送给父Agent。禁止轮询（"do NOT call sessions_list or any polling tool"），因为轮询浪费API调用且延迟大。交付错误分为瞬态（网络问题，自动重试）和永久（用户删除聊天，不重试）。

**关键源码位置**：
- 子Agent生成：`src/agents/subagent-spawn.ts`
- 深度计算：`src/agents/subagent-depth.ts:124-176`
- 结果宣告：`src/agents/subagent-announce.ts:82-85`
- 注册表管理：`src/agents/subagent-registry.ts:65-99`
- 子Agent控制：`src/agents/subagent-control.ts:45-52`

```typescript
// OpenClaw的子Agent编排：像一家有项目调度和交付系统的跨国集团

class OpenClawSubagentSystem {
    async spawnSubagent(parentSession, goal) {
        // ① 深度守卫：递归计算spawn链深度
        const depth = getSubagentDepthFromSessionStore(parentSession.key);
        if (depth >= this.config.maxSpawnDepth) {
            throw new Error("Max subagent depth exceeded");
        }
        
        // ② 创建独立session（隔离上下文）
        const childSession = await createSession({
            spawnedBy: parentSession.key,
            spawnDepth: depth + 1,
            workspace: inheritWorkspace(parentSession),
        });
        
        // ③ 注册到全局注册表
        this.registry.register(childSession);
        
        // ④ 子Agent独立运行（异步，不阻塞父Agent）
        runChildAgent(childSession, goal);
        
        // 父Agent立即返回，继续自己的工作
        return { sessionKey: childSession.key, status: "running" };
    }
    
    async onChildComplete(childSession, result) {
        // ⑤ Push模式交付：子Agent主动推送结果
        const parentSession = this.registry.getParent(childSession);
        
        const deliveryResult = await this.announce({
            target: parentSession,
            payload: result,
            maxRetries: 3,
            timeoutMs: 120_000,
        });
        
        if (deliveryResult.transientError) {
            // 瞬态错误：自动重试
            await this.scheduleRetry(childSession, result);
        } else if (deliveryResult.permanentError) {
            // 永久错误：记录失败，不再重试
            await this.recordPermanentFailure(childSession);
        }
    }
    
    async steerSubagent(sessionKey, message) {
        // ⑥ 向运行中的子Agent发送 steer 指令
        // 有字符限制(4K)和速率限制(2秒间隔)
        return this.control.send(sessionKey, message);
    }
}
```

### OpenCode："子Agent类型化+权限继承"的内联执行

**设计定位**：子Agent是类型化的专用Agent（general/explore）、权限继承+限制、支持内联执行——适合IDE集成、需要精细分工的场景。

> 💡 **名词解释：Task Tool（任务工具）**
>
> 通俗地说：设计工作室里有专门的"调研专员"和"代码审查专员"——调研专员只负责查资料不写代码，代码审查专员只读不改。主管根据任务类型指派不同的人。
>
> 技术定义：OpenCode的`TaskTool`创建child session并在其中运行特定类型的子Agent（如`explore`只读探索、`general`通用任务）。子Agent继承父会话权限，但默认禁用`todowrite`/`todoread`（防止修改父会话的todo列表），且默认禁止再创建子Agent。

> 💡 **名词解释：@agent 语法**
>
> 通俗地说：小李对助手说"@调研专员 帮我查一下这个API的文档"——这是一个显式的指令，告诉系统"我要调用专门的调研人员"。
>
> 技术定义：OpenCode支持用户在消息中使用`@agent`语法显式调用子Agent。此时系统自动追加task工具调用提示，并跳过权限检查（因为用户已明确授权）。

**关键源码位置**：
- TaskTool执行：`packages/opencode/src/tool/task.ts:45-102`
- Subtask内联执行：`packages/opencode/src/session/prompt.ts:353-527`
- 子Agent定义：`packages/opencode/src/agent/agent.ts:115-156`
- @agent触发：`packages/opencode/src/session/prompt.ts:1263-1285`

```typescript
// OpenCode的子Agent编排：像一个有专业分工的设计工作室

class OpenCodeSubagentSystem {
    async executeTask(params, ctx) {
        // ① 权限检查（@agent调用时跳过）
        if (!ctx.extra?.bypassAgentCheck) {
            await ctx.ask({ permission: "task", patterns: [params.subagent_type] });
        }
        
        const agent = await Agent.get(params.subagent_type);
        
        // ② 创建child session，继承父session但限制权限
        const session = await Session.create({
            parentID: ctx.sessionID,
            title: params.description + ` (@${agent.name} subagent)`,
            permission: [
                { permission: "todowrite", pattern: "*", action: "deny" },  // 不能改父todo
                { permission: "todoread", pattern: "*", action: "deny" },
                { permission: "task", pattern: "*", action: "deny" },      // 默认禁止递归
            ],
        });
        
        // ③ 在child session中运行子Agent
        const result = await SessionPrompt.prompt({
            sessionID: session.id,
            agent: agent.name,           // general | explore | compaction
            tools: { todowrite: false, todoread: false, task: false },
        });
        
        return {
            task_id: session.id,
            output: `<task_result>${result.text}</task_result>`,
        };
    }
    
    async handleInlineSubtask(task, ctx) {
        // ④ 内联执行：主循环直接检测pending subtask，无需LLM再生成tool-call
        const taskTool = await TaskTool.init();
        const result = await taskTool.execute(task.args, {
            ...ctx,
            extra: { bypassAgentCheck: true },  // 系统触发的，跳过权限
        });
        
        // 插入synthetic user message兼容Gemini的消息序列要求
        if (task.command) {
            await Session.updateMessage({ role: "user", content: "Summarize the task output..." });
        }
        
        return result;
    }
}
```

### 四项目核心差异对比表

| 维度 | Hermes Agent | nanobot | OpenClaw | OpenCode |
|------|-------------|---------|----------|----------|
| **核心策略** | 完整实例+同步阻塞并发委托 | 后台异步+消息注入 | 事件驱动异步委派 | 类型化子Agent+内联执行 |
| **子Agent形态** | 完整AIAgent实例（再走run_conversation） | 独立asyncio.Task + 精简ToolRegistry | 独立session + 独立执行循环 | child session + 专用agent类型 |
| **执行模式** | 同步阻塞（父等子） | 后台异步（父不等子） | 后台异步（父不等子） | 内联同步（主循环直接执行） |
| **并发能力** | ThreadPoolExecutor（多子并行） | 单个子Agent后台运行 | 支持多子并行 | 串行执行 |
| **结果交付** | JSON摘要直接返回（tool result） | system消息注入Bus | Push-based announce（主动推送） | 直接返回task结果 |
| **深度限制** | 运行时递归计算 + 全局开关 | 未明确限制 | 递归遍历spawnedBy链 | 默认禁用子Agent的task权限 |
| **角色分层** | leaf vs orchestrator | 无 | 无 | general vs explore vs compaction |
| **中断机制** | 级联interrupt（父→子→孙） | cancel_by_session（按会话） | steer + abort | AbortController |
| **工具继承** | 父子交集 - 黑名单 | 独立Registry（去掉message/spawn） | 继承父工具集 | 继承 + 权限限制 |
| **特殊设计** | 父子文件交叉提醒、审批回调注入防死锁 | 15轮限制（主循环40轮） | 瞬态/永久错误分类、steer控制 | @agent语法、subtask内联 |

---

## 11.5 独特高价值设计：各项目的闪光点

### Hermes Agent：父子文件交叉提醒——"同事改了你看过的文件"

`tools/delegate_tool.py:1462-1733`：在子Agent启动时快照父Agent当前已读路径，结束时反查"这些路径上有谁在快照之后写过"——有就把警告塞进summary。

**解决的问题**：父Agent的`file_state`缓存让它"以为自己手里有文件最新内容"，但子Agent可能修改了它。如果父Agent后续直接`edit`那个文件，会撞到stale-content检查并失败。

**什么场景值得借鉴**：任何父子Agent共享文件系统的场景。

### Hermes Agent：审批回调注入——"防止子线程死锁父TUI"

`tools/delegate_tool.py:1471-1479`：在`ThreadPoolExecutor(initializer=...)`时给每个工人线程预装一个非交互式的审批回调，避免子Agent的危险命令审批回退到`input()`而死锁父Agent的`prompt_toolkit` TUI。

**解决的问题**：父Agent的TUI持有stdin，子Agent工人线程没有继承父线程的TLS审批回调，回退到`input()`后stdin已被占用——死锁。

**什么场景值得借鉴**：任何使用TUI + 多线程 + 交互式审批的Agent系统。

### nanobot：system消息注入——"结果交给主Agent总结"

`nanobot/agent/subagent.py:168-197`：子Agent结果以`channel="system"`注入Bus。`loop.py:364-382`中system消息被特殊处理：不走`_save_turn()`（不污染用户历史），直接触发主Agent生成自然语言总结。

**解决的问题**：如果子Agent直接给用户发消息，用户会收到"不知道从哪来的"通知；如果作为user消息保存，会虚假记录"用户输入"。

**什么场景值得借鉴**：任何后台任务完成后需要"优雅回到主对话"的Agent系统。

### nanobot：独立ToolRegistry——"最小权限的沙箱"

`nanobot/agent/subagent.py:92-108`：子Agent新建独立的`ToolRegistry()`，只注册filesystem、shell、web工具——明确排除`MessageTool`（防止骚扰用户）和`SpawnTool`（防止递归）。

**解决的问题**：子Agent如果保留所有工具，可能直接给用户发消息或无限递归生成子子Agent。

**什么场景值得借鉴**：所有需要限制子Agent能力的系统。

### OpenClaw：Announce Push模式——"禁止轮询"

`src/agents/subagent-spawn.ts:82-85`：明确注释"Auto-announce is push-based. Do NOT call sessions_list, sessions_history, exec sleep, or any polling tool."。

**解决的问题**：轮询浪费API调用、增加复杂性、延迟结果接收。Push模式让子Agent完成时主动通知父Agent。

**什么场景值得借鉴**：任何异步任务结果交付的系统。

### OpenClaw：深度守卫的递归计算

`src/agents/subagent-depth.ts:124-176`：不仅读取显式存储的`spawnDepth`，还递归遍历`spawnedBy`链计算深度，并做防循环检测（`visited` Set）。

**解决的问题**：session store可能在不同进程中被访问，深度信息可能不完整。多种计算方式+防循环确保深度限制始终有效。

**什么场景值得借鉴**：任何需要严格防止无限递归的系统。

### OpenCode：子Agent类型化——"不同的人做不同的事"

`packages/opencode/src/agent/agent.ts:115-156`：`explore`agent只读不写（禁用edit/write），`general`agent保留编辑权限。每种类型有独立的prompt和权限配置。

**解决的问题**：单一Agent能力过载、prompt冗长冲突。专用子Agent只做一件事，能力边界清晰。

**什么场景值得借鉴**：任何需要处理多种任务类型的复杂Agent系统。

### OpenCode：Subtask内联执行——"不用LLM再决定一次"

`packages/opencode/src/session/prompt.ts:353-527`：主循环直接检测pending subtask part并实例化`TaskTool`执行，无需LLM再次生成tool-call。

**解决的问题**：如果让LLM决定"要不要调用task工具"，会增加一次LLM调用开销和延迟。系统已经知道有subtask要处理，直接执行更高效。

**什么场景值得借鉴**：任何已知需要执行子任务、无需LLM再决策的场景。

---

## 11.6 设计思想总结

如果我要自己构建一个Agent系统，子Agent编排部分的设计checklist是什么？

### 必须做（底线）

| Check | 项目 | 说明 |
|-------|------|------|
| ☐ | 任务分解接口 | 父Agent能明确触发子任务（工具调用或特殊消息） |
| ☐ | 上下文隔离 | 子Agent有独立的messages/会话，不污染父上下文 |
| ☐ | 结果回传 | 子Agent完成后能把结果交付给父Agent |
| ☐ | 深度限制 | 最多2-3层嵌套，防止无限递归 |

### 视场景而定（加分项）

| 场景 | 推荐设计 | 参考项目 |
|------|---------|---------|
| 需要并行外包复杂子任务 | 完整AIAgent实例 + ThreadPoolExecutor并发 | Hermes |
| 个人助手/后台任务 | 后台asyncio.Task + system消息注入 | nanobot |
| 7×24生产级网关 | 独立session + push-based announce + steer控制 | OpenClaw |
| IDE集成/精细分工 | 类型化子Agent + 权限继承限制 + @agent语法 | OpenCode |
| 父子共享文件系统 | 文件交叉提醒（子改父读过的文件时通知） | Hermes |
| TUI + 多线程审批 | 工人线程预装审批回调防死锁 | Hermes |
| 已知子任务无需LLM决策 | 内联执行（主循环直接处理） | OpenCode |

### 最重要的三条经验

1. **"子Agent是实例，不是prompt段"**：Hermes让每个子Agent完整走一遍`run_conversation()`，所有横切关注点（中断、配额、审批、可观测性）自动复用父Agent的基础设施。这比"在父Agent的prompt里加一段'请假装你是另一个Agent'"可靠得多。

2. **"同步阻塞"和"后台异步"是两种截然不同的哲学**：Hermes选择同步阻塞——换来父子状态一致性和中断点可预测；nanobot和OpenClaw选择后台异步——换来用户体验（不阻塞交互）。没有绝对优劣，取决于场景。如果子任务的结果是父Agent继续推理的必要前提，选同步；如果子任务可以"完成后通知"，选异步。

3. **"结果交付方式决定了系统的耦合度"**：nanobot的system消息注入、OpenClaw的push-based announce、Hermes的JSON直接返回——三种交付方式对应三种耦合度。Bus注入最解耦（子Agent不需要知道父Agent的存在），push模式最可靠（主动通知不依赖父Agent查询），直接返回最同步（父Agent当场拿到结果）。

---

## 11.7 本章小结

> **实习生小李说：**
>
> "一个人的力量有限，任务太大时就得请同事帮忙。但请同事不是'把资料一丢就不管了'——要明确分工、要给独立空间、要知道他们做完了、还要防止无限分包。
>
> "Agent的子Agent编排也一样：上下文隔离是'独立工位'，结果回传是'交报告'，深度限制是'最多两层分包'。最关键的是：子Agent是完整的'另一个人'，不是'我在脑子里分裂出的第二人格'。
>
> "好了，我的团队协作搞清楚了。下一章，我们来看看实习生上班第一天需要准备什么——初始化与环境。"
