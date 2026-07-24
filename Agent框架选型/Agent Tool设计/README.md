---
title: Agent Tool 设计专题
date: 2026-07-24
tags:
  - AI-Agent
  - tool-calling
  - MCP
aliases:
  - Agent 工具专题
---

# Agent Tool 设计专题

本目录研究 Agent Tool 的创建方式、运行形态、协议边界和生产治理。

## 文档

- [[Agent Tool创建与形态调研|Agent Tool 创建与形态调研]]：回答 Tool 是否只是“函数 + MCP”，建立声明方式、执行位置、能力形态三套分类，并给出已有函数接入 Tool、注解与反射生成 Schema、模型 Tool Call 和 Runtime 调度的完整实现链路。

## 核心结论

```text
模型看到的 Tool
  = name + description + input schema + result contract

Tool 的执行端
  = 本地函数 / 远程 API / MCP Server / 模型厂商平台 / 沙箱 / 工作流 / 另一个 Agent

MCP
  = Tool 的标准化发现与调用协议
  != Tool 的唯一实现方式
```
