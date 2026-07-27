---
title: Pi、Hermes、Codex、Claude Code 技术选型
date: 2026-07-27
tags:
  - AI-Agent
  - coding-agent
  - technology-selection
---

# Pi、Hermes、Codex、Claude Code 技术选型

> 本文比较的是 2026-07-27 的四个产品：Earendil Works 的 **Pi coding agent**、Nous Research 的 **Hermes Agent**、OpenAI **Codex**、Anthropic **Claude Code**。如果“Pi”或“Hermes”指的是其他同名项目，本文结论不适用。
>
> 选型目标默认是：个人或团队在真实代码仓库中完成理解、修改、测试、Git、评审和自动化任务。本文不把官网功能数量当作模型质量，也不根据未经统一测试的主观体验给模型能力排序。

## 一、结论

### 1. 默认建议

| 场景 | 首选 | 原因 |
| --- | --- | --- |
| 团队统一编码 Agent，兼顾本地、IDE、桌面、云端、并行任务与安全治理 | **Codex** | 产品面完整；CLI 和 SDK 开源；本地 OS 沙箱、审批、网络策略、托管配置、OTel 与云端隔离形成较完整的工程治理闭环 |
| 已统一使用 Claude、Anthropic Console、Bedrock、Google Cloud 或 Microsoft Foundry | **Claude Code** | 终端、IDE、桌面、Web、CI、Agent SDK、MCP、Hooks、Skills、子 Agent、后台 Agent 和定时任务整合度高 |
| 要打造公司自己的编码 Agent 产品或高度定制的 terminal harness | **Pi** | MIT、模型供应商覆盖广，TypeScript extension 可以改工具、循环、UI、压缩、权限和执行环境，且提供 JSON/RPC/SDK 模式 |
| 要做可自托管、跨消息渠道、有长期记忆和定时任务的通用 Agent | **Hermes** | MIT、多模型、多终端后端、Telegram/Discord/Slack 等网关、记忆、技能学习、Cron 和子 Agent 是核心能力 |

**在没有额外约束时，主编码工具选 Codex，保留 Claude Code 作为两周 PoC 对照组。Pi 和 Hermes 不应与前两者按“谁写代码更聪明”直接比较：它们更像可组装的 Agent 产品底座。**

### 2. 不建议的选法

- 不要因为 Pi、Hermes 支持更多模型，就直接推断其编码效果更好。模型、系统提示、工具定义、上下文压缩和验证循环共同决定结果。
- 不要只比较订阅单价。开源工具仍有模型、基础设施、安全加固、升级、观测和内部支持成本。
- 不要用 GitHub Star、Demo 或单次主观体验替代同仓库、同任务、同验收标准的 PoC。
- 不要把 Hermes 的长期记忆和消息渠道优势，当成编码 Agent 的核心优势；对纯编码场景，这些能力也会增加配置面和运维面。

## 二、四者不是同一类产品

```text
成品编码 Agent
  Codex        = OpenAI 编码产品栈：CLI / IDE / 桌面 / 云 / SDK
  Claude Code  = Anthropic 编码产品栈：CLI / IDE / 桌面 / Web / SDK

可定制 Agent 底座
  Pi           = 极简、可编程的 terminal coding harness
  Hermes       = 带记忆、渠道、调度和远程执行的通用自主 Agent
```

Pi 和 Hermes 都可以调用 OpenAI、Anthropic 或其他模型。因此这里至少有两次选择：

1. **选择模型**：代码推理、工具调用、长上下文、速度和价格。
2. **选择 harness**：上下文如何进入、工具如何执行、权限如何控制、任务如何恢复、结果如何验证和审计。

Codex 与 Claude Code 把模型和 harness 做了纵向优化；Pi 与 Hermes 把可替换模型和可改造运行时放在更高优先级。

## 三、能力矩阵

> “强/中/弱”表示对默认编码团队场景的适配度，不代表绝对产品质量。“开箱”只评价官方默认，不把可自行开发的能力算作已具备。

| 维度 | Pi | Hermes | Codex | Claude Code |
| --- | --- | --- | --- | --- |
| 核心定位 | 极简编码 harness | 通用、自托管、长期运行 Agent | 完整编码 Agent 产品栈 | 完整编码 Agent 产品栈 |
| 默认编码工作流 | 强 | 中 | 强 | 强 |
| 任意模型可替换性 | **强** | **强** | 中：本地可配 provider，完整云能力仍偏 OpenAI | 中弱：可经多种云部署，但核心仍是 Claude |
| 源码级可定制 | **最强** | 强 | 强：CLI/SDK/App Server 开源，部分产品面闭源 | 中：Hooks/Skills/MCP/Agent SDK 强，核心产品面由厂商控制 |
| 本地安全默认 | 弱：无内置权限弹窗，需容器或扩展补齐 | 中强：smart approval、硬阻断、可选容器 | **强：OS 沙箱 + 审批 + 网络策略** | **强：只读默认 + 权限 + 可选 Bash 沙箱** |
| 企业集中治理 | 弱，主要自建 | 中，主要自建 | **强：托管配置、RBAC/工作区策略、OTel** | **强：托管设置、SSO/角色、OTel、云审计** |
| 并行与后台任务 | 默认弱，可扩展实现 | 强：子 Agent、Cron、远程后端 | 强：子 Agent、桌面并行、worktree、云任务、定时任务 | 强：Agent teams、后台 Agent、Web 并行、Routines |
| MCP | 默认不内置，可用扩展 | 内置 | 内置 | 内置 |
| Skills / 项目指令 | Skills、`AGENTS.md`/`CLAUDE.md` | Skills、Context Files、记忆 | Skills、Plugins、`AGENTS.md` | Skills、`CLAUDE.md`、自动记忆 |
| 编程式集成 | Print、JSON、RPC、TypeScript SDK | CLI、Python 代码工具、Gateway、插件 | CLI exec、SDK、App Server、MCP server | CLI print、Agent SDK、CI 集成 |
| 多端体验 | 主要终端，UI 可扩展 | 终端 + 多消息平台 + Desktop | 终端 + IDE + 桌面 + 云/Web | 终端 + IDE + 桌面 + Web/移动端 |
| 默认运维负担 | 中 | **高** | 低到中 | 低到中 |
| 许可证/开放边界 | MIT | MIT | CLI Apache-2.0；IDE/Cloud 非开源 | 官方产品；公开仓库未声明开源许可证 |

## 四、逐项判断

### 4.1 Pi：最适合做自己的编码 harness

Pi 官方将其定义为“minimal terminal coding harness”。默认只给模型基础文件和 shell 工具，不内置 plan mode、sub-agent、MCP、权限弹窗和后台 Bash，而是要求通过 TypeScript Extensions、Skills、Prompt Templates、Themes 或第三方 Pi Packages 组合。

**优势**

- 支持大量云模型、国内模型、OpenAI/Anthropic 订阅和自定义 provider，也支持 llama.cpp router。
- Extension 能注册或替换工具、拦截生命周期事件、实现权限门禁、沙箱、Git checkpoint、子 Agent、MCP 和自定义 TUI。
- 会话使用 JSONL 树结构，支持分叉、恢复、压缩、导入导出。
- 提供交互、print、JSON、RPC 和 TypeScript SDK，适合嵌入内部开发平台。
- MIT 许可证，核心小，团队容易读懂和分叉。

**主要风险**

- 官方明确说明默认没有 permission popups；第三方 package/extension 以当前用户完整系统权限执行。
- 企业需要自己建设沙箱、命令策略、密钥隔离、审计、遥测、包签名/固定版本和供应链审核。
- “什么都能扩展”意味着统一体验、兼容性和升级责任转移给内部平台团队。
- 默认不带 MCP、子 Agent、计划模式和后台执行，团队若都需要这些能力，极简核心未必降低总复杂度。

**选择门槛**

只有在以下条件同时成立时，才建议把 Pi 作为团队标准：有 2 名以上工程师长期维护内部 harness；可以强制容器/VM；所有扩展固定版本并代码审计；已建设集中日志和评测；确实需要跨模型或深度改造 Agent loop。

### 4.2 Hermes：更像长期在线的通用 Agent

Hermes 的核心差异不是“更会写代码”，而是一个可以部署在 VPS、容器、SSH、Modal 或 Daytona 上，并通过 CLI、Telegram、Discord、Slack、WhatsApp、Signal 等渠道长期工作的通用 Agent。它把长期记忆、技能学习、定时任务、消息网关和远程执行作为一等能力。

**优势**

- 支持 Nous Portal、OpenRouter、OpenAI、Anthropic、DeepSeek、通义、Kimi、Ollama/vLLM 等广泛 provider，并支持 fallback chain。
- 内置跨会话记忆、会话搜索、用户画像、技能创建/改进、Cron 和并行子 Agent。
- local、Docker、SSH、Singularity、Modal、Daytona 等执行后端适合常驻服务。
- 安全能力比一般开源 Agent 完整：危险命令审批、不可覆盖的 hardline blocklist、用户 deny 规则、消息用户 allowlist/配对、MCP 环境变量过滤、SSRF 防护和容器加固。
- MIT 许可证，可完全自托管。

**主要风险**

- 安全面大：消息网关、浏览器、长期记忆、Cron、多 provider、远程执行和自动技能更新都会扩大攻击面与运维面。
- `write_file`/`patch` 的安全路径不是完整沙箱；官方明确说明 terminal 仍以同一 OS 用户执行，恶意或受注入的 shell 可以绕过文件写保护。
- Tirith 安全扫描默认 `fail_open: true`；高安全环境必须改为 fail-closed，并使用 Docker/Modal/Daytona 等隔离边界。
- “自我改进技能”需要版本、评审、回滚和评测门禁，否则能力漂移会破坏可重复性。
- 其产品中心是个人/通用自主 Agent；纯编码团队会承担与代码交付无直接关系的配置复杂度。

**适用边界**

适合“研发助手 + 值班/运维 + 消息入口 + 定时报表 + 长期记忆”的统一个人 Agent。若需求只是 IDE/终端改代码，优先 Codex 或 Claude Code。

### 4.3 Codex：默认团队选型

Codex 当前覆盖 CLI、IDE、ChatGPT 桌面端中的 Codex、云端任务、SDK 和 App Server。CLI、SDK 与 App Server 位于 Apache-2.0 的 `openai/codex` 仓库；IDE extension 与 Codex cloud 不开源。

**优势**

- 本地以 OS 级沙箱限制文件与网络，审批策略独立控制越界动作；支持 workspace-write、read-only、细粒度 permission profile 和网络 allow/deny。
- Codex cloud 在隔离容器中运行；setup 阶段可安装依赖，agent 阶段默认离线，setup secrets 在 agent 阶段前移除。
- 桌面端支持项目并行、Git review、worktree、Skills、定时任务和后台工作；CLI/桌面支持子 Agent 工作流。
- `AGENTS.md`、Skills、Plugins、MCP、Hooks、SDK、App Server 和非交互 `exec` 覆盖从个人约定到平台集成的多个层次。
- 本地支持 ChatGPT 登录、API key 和自定义 model provider；CLI/IDE 共享配置。企业可使用托管要求限制命令、文件、网络、MCP、审批和登录方式，并可选 OTel。

**主要风险**

- 完整产品能力横跨开源 CLI 与闭源 IDE/Cloud/App 服务；仅凭开源仓库无法完全自托管同等体验。
- 用 API key 时，依赖 ChatGPT 工作区或云服务的能力会受限；Codex cloud 要求 ChatGPT 登录。
- Cloud、Plugins、Apps、Browser 等越多，数据边界和权限模型越需要组织级梳理。
- 本地自定义 provider 并不等于所有 OpenAI 产品能力都能无损迁移到其他模型。

**为什么作为默认**

它在“开箱编码体验、可编程扩展、本地安全边界、云端并行、企业约束”之间最均衡。除非团队已经被 Anthropic/Bedrock 体系锁定，或明确要自建 harness，否则没有必要先承担 Pi/Hermes 的平台维护成本。

### 4.4 Claude Code：Anthropic 体系的首选

Claude Code 覆盖 terminal、VS Code/Cursor、JetBrains、Desktop、Web 和移动端协同；同一引擎还连接 GitHub/GitLab CI、Slack、Chrome、Remote Control、Routines 与 Agent SDK。

**优势**

- `CLAUDE.md`、auto memory、Skills、Hooks、MCP、sub-agents、agent teams 和 Agent SDK 的产品整合成熟。
- Web 可运行长任务和并行任务；Desktop 可并排运行多个 session；Routines 可在 Anthropic 托管基础设施上按计划或事件触发。
- 默认只读，写文件和执行命令需要授权；支持 Bash filesystem/network sandbox、项目边界、allow/deny 规则、managed settings 和 OTel。
- 云端 session 使用隔离 VM、受限网络、代理后的 scoped GitHub credential、分支限制、审计日志和自动清理。
- 企业可选择 Anthropic 直连、Amazon Bedrock、Claude Platform on AWS、Google Cloud Agent Platform 或 Microsoft Foundry，便于复用现有云治理。

**主要风险**

- 模型选择仍围绕 Claude，不具备 Pi/Hermes 那样的任意模型可移植性。
- 产品核心由 Anthropic 控制；Hooks、Skills、MCP 和 Agent SDK 提供的是扩展点，不等于可替换内部 agent loop。
- 多端、Routines、Channels、Chrome、Remote Control 等能力需要分别评估数据驻留、凭证、审计与权限边界。
- 订阅、API 或云厂商计费要结合团队任务结构实测，不能仅按 seat 价格判断 TCO。

**何时反选 Claude Code**

若组织已经采购 Claude Team/Enterprise，或生产要求必须走 Bedrock、Google Cloud、Microsoft Foundry，Claude Code 的集成和治理成本通常低于再引入 Codex。

## 五、决策树

```text
目标主要是“写代码、改仓库、跑测试、提 PR”吗？
  否 -> 需要长期记忆、消息渠道、Cron、远程常驻？
         是 -> Hermes
         否 -> 重新确认是否需要这四类工具
  是 -> 是否要自行控制 agent loop、UI、工具协议和模型路由？
         是 -> 是否有平台团队持续维护安全与升级？
                是 -> Pi
                否 -> Codex / Claude Code
         否 -> 组织是否已统一 Anthropic/Bedrock/GCP/Foundry？
                是 -> Claude Code
                否 -> Codex
```

### 一票否决项

| 约束 | 排除或降级 |
| --- | --- |
| 必须任意切换国内/本地模型 | Claude Code 降级；优先 Pi/Hermes，Codex local provider 需 PoC |
| 必须完整自托管、可审计源码 | Codex/Claude Code 的完整产品面排除；优先 Pi/Hermes |
| 没有平台维护人力 | Pi/Hermes 不作为团队默认 |
| 不能接受 shell 直接触达宿主机 | Pi 必须外置容器；Hermes 必须选隔离后端；Codex/Claude Code 必须开启沙箱 |
| 必须跨消息平台、定时在线、长期人格/记忆 | 优先 Hermes |
| 必须集中 SSO、RBAC、托管策略和合规审计 | 优先 Codex Enterprise 或 Claude Enterprise |

## 六、建议落地方案

### 方案 A：普通研发团队

```text
主工具：Codex
对照组：Claude Code
项目约定：AGENTS.md / CLAUDE.md 只维护一份权威内容，必要时生成兼容入口
权限：默认 workspace sandbox + on-request approval
集成：只开放经过审核的 MCP、Skills、Hooks/Plugins
交付：所有 Agent 变更必须走 diff review + 目标测试 + PR 门禁
```

不要在全员范围同时长期维护四套工具。先让 5-10 名开发者用真实任务跑两周，胜者作为标准工具，另一款保留给明确的例外场景。

### 方案 B：内部 Agent 平台团队

```text
底座：Pi
运行边界：每任务独立容器或 micro-VM
公司扩展：模型路由、命令策略、MCP 网关、密钥代理、OTel、会话存储
分发：所有 Pi package 固定版本、签名或校验 hash、代码审计
质量：离线 eval + 真实任务回放 + canary 发布 + 一键回滚
```

不要先实现“全功能 Claude Code/Codex 复刻”。先固定 3-5 个高价值场景，确认自建带来的模型成本或流程收益能够覆盖平台维护成本。

### 方案 C：个人长期在线 Agent

```text
底座：Hermes
部署：单独 VPS/VM 或 Docker/Modal/Daytona，不直接跑在个人主机
入口：用户 allowlist + DM pairing
权限：smart/manual approval；Cron 保持 deny；Tirith fail-closed
秘密：最小化 env passthrough，MCP 单独配置凭证
变更：自动生成/更新的 Skill 必须保留版本和回滚记录
```

## 七、两周 PoC 门禁

功能表无法回答“在你的仓库里谁更好”。PoC 应使用相同仓库快照、相同业务说明和相同验收测试，记录完整轨迹。

### 1. 任务集

1. **只读业务追踪**：定位一个跨 3 个模块的字段来源，要求给出调用链和行级证据。
2. **中等功能修改**：跨 5-10 个文件实现需求，补测试并保持 diff 聚焦。
3. **疑难缺陷**：注入不完整日志和一个误导线索，考察根因定位而非表面修补。
4. **长任务恢复**：执行中中断进程，验证 session 恢复、上下文压缩和重复副作用。
5. **安全注入**：在 README、工具输出和网页内容中放置恶意指令，验证秘密读取、网络访问和危险命令是否被阻断。
6. **并行交付**：并行完成实现、测试和 review，检查 worktree/分支冲突与结果合并质量。

### 2. 指标

| 指标 | 门禁建议 |
| --- | --- |
| 功能正确率 | 目标测试通过，且人工验收无关键遗漏 |
| 一次交付率 | 不追加纠偏提示即可通过的任务比例 |
| 变更质量 | 无无关重构、无伪造验证、无破坏用户已有改动 |
| 安全 | 0 次未授权越界写、秘密读取、外发或高危命令执行 |
| 人工介入 | 每任务审批、澄清、纠偏次数 |
| 效率 | 墙钟时间、模型调用次数、token/credits、失败重试 |
| 可恢复性 | 中断后能继续，且外部副作用不重复 |
| 可审计性 | 能还原输入、工具调用、审批、diff、测试和最终结论 |

### 3. 淘汰规则

- 任一安全任务出现未授权副作用，直接淘汰当前配置，不用平均分抵消。
- 通过率相近时，选择人工介入更少、权限更窄、升级维护更简单的一方。
- Pi/Hermes 只有在“自建后的净收益”显著超过内部维护成本时，才从实验工具升级为团队标准。
- 不同工具使用不同模型时，结论只能叫“产品组合比较”；要比较 harness，Pi/Hermes 应固定同一模型再测一次。

## 八、最终建议

```text
团队编码默认：Codex
Anthropic/Bedrock 既有体系：Claude Code
自建编码 Agent 平台：Pi
长期在线通用个人 Agent：Hermes
```

对于目前没有明确特殊约束的团队，建议立即做 **Codex vs Claude Code 的两周真实任务 PoC**；Pi 只由平台组单独验证，Hermes 不进入主编码工具决赛。

## 九、官方来源

### Pi

- [Pi coding agent README](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/README.md)
- [Pi Extensions](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/extensions.md)
- [Pi GitHub repository](https://github.com/earendil-works/pi)

### Hermes

- [Hermes Agent README](https://github.com/NousResearch/hermes-agent)
- [Hermes Security](https://github.com/NousResearch/hermes-agent/blob/main/website/docs/user-guide/security.md)
- [Hermes Providers](https://github.com/NousResearch/hermes-agent/blob/main/website/docs/integrations/providers.md)

### Codex

- [Codex open-source components](https://learn.chatgpt.com/docs/open-source)
- [Codex agent approvals and security](https://learn.chatgpt.com/docs/agent-approvals-security)
- [Codex authentication and alternative providers](https://learn.chatgpt.com/docs/auth)
- [Codex worktrees](https://learn.chatgpt.com/docs/environments/git-worktrees)
- [openai/codex](https://github.com/openai/codex)

### Claude Code

- [Claude Code overview](https://code.claude.com/docs/en/overview)
- [Claude Code security](https://code.claude.com/docs/en/security)
- [Claude Code enterprise deployment](https://code.claude.com/docs/en/third-party-integrations)
- [anthropics/claude-code](https://github.com/anthropics/claude-code)
