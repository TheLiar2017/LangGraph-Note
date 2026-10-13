---
aliases: [首页, Home, 主页]
tags: [index, MOC]
cssclass: home
---

# 🚀 LangGraph 学习笔记

> **LangGraph** 是 LangChain 生态中的低层级编排框架，用于构建有状态、可持久化的 AI Agent 工作流。
> 基于**图（Graph）**架构，支持循环、分支、人工介入和长期记忆。

---

## 📚 学习路线

### ⚙️ 第零步
- [[00-环境配置与安装]] — 安装 LangGraph + PyCharm 环境配置

### 🌱 基础入门
- [[概念-状态图-StateGraph]] — StateGraph 核心概念
- [[概念-节点与边]] — Node、Edge、Conditional Edge
- [[概念-状态管理]] — TypedDict、Annotated、reducer
- [[概念-消息与记忆]] — MessagesState、对话历史管理

### 🔧 核心功能
- [[功能-工具调用-Tool-Calling]] — 绑定工具、ToolNode
- [[功能-流式输出-Streaming]] — stream / astream 用法
- [[功能-检查点-Checkpoint]] — 持久化与恢复
- [[功能-人工介入-Human-in-the-Loop]] — interrupt、审批流程

### 🤖 Agent 构建
- [[实战-React-Agent]] — 用 create_react_agent 快速搭建
- [[实战-自定义Agent]] — 手动构建 StateGraph Agent
- [[实战-多Agent协作]] — Supervisor / Hierarchical 模式
- [[实战-RAG-Agent]] — 检索增强生成 Agent

### 🔌 模型接入
- [[模型-DeepSeek]] — DeepSeek API 接入
- [[模型-本地模型-Ollama]] — 本地模型接入
- [[模型-OpenAI-兼容服务]] — 其他 OpenAI 兼容 API

---

## 📝 学习日志

| 日期 | 内容 | 状态 |
|:----:|------|:----:|
| 2026-10-13 | 流式输出 Streaming 文档 | ✅ |
| 2026-10-12 | 工具调用 Tool Calling 文档 | ✅ |
| 2026-10-12 | 消息与记忆、状态管理、节点与边、StateGraph 文档 | ✅ |
| 2026-10-12 | 项目初始化，环境配置 | ✅ |

---

## 🔗 参考资源

- 📖 [LangGraph 官方文档](https://langchain-ai.github.io/langgraph/)
- 📖 [LangChain 文档](https://docs.langchain.com/)
- 💻 [LangGraph GitHub](https://github.com/langchain-ai/langgraph)
- 🎓 [LangGraph Academy 课程](https://academy.langchain.com/courses/intro-to-langgraph)
- 🤖 [DeepSeek API 文档](https://platform.deepseek.com/api-docs)

---

## 🏷️ 标签索引

- `#概念` — 核心概念笔记
- `#功能` — 功能模块详解
- `#实战` — 实战项目记录
- `#模型` — 模型接入配置
- `#问题` — 踩坑与解决方案