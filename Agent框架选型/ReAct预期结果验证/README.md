---
title: ReAct 预期结果验证专题
date: 2026-07-24
tags:
  - AI-Agent
  - ReAct
  - verification
aliases:
  - ReAct 验证专题
---

# ReAct 预期结果验证专题

本目录汇总 ReAct Agent 循环中的结果验证与人工介入设计。

## 文档

- [[ReAct循环中的预期结果验证设计|ReAct 循环中的预期结果验证设计]]：定义 Expected Result 验证契约、Verifier 路由、三态判断、定向重试、两级验证和 Judge 校准。
- [[无对话框Agent的人工介入交互设计|无对话框 Agent 的人工介入交互设计]]：定义没有聊天窗口时，Agent 如何通过异步任务、结构化处理单和通知渠道暂停并恢复执行。

## 关系

```text
Result Verifier
  -> PASS：继续执行或完成
  -> FAIL：重试、重规划或终止
  -> UNKNOWN / 需要审批
       -> 创建 InteractionRequest
       -> Agent 暂停并保存 checkpoint
       -> 用户通过任务中心或通知卡片处理
       -> Agent 恢复执行
```
