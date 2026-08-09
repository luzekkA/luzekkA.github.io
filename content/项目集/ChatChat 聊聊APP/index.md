---
title: ChatChat 聊聊APP
date: 2026-08-05T12:00:00+08:00
draft: false
toc: true
tags:
  - React Native
  - AI
  - RAG
  - 移动开发
  - Expo
categories:
  - 项目集
summary: 单机优先、零自建服务器成本、隐私本地化的 AI 记忆助手与理性咨询 App，将人生经历沉淀为本地可检索的长期语义记忆。
---

# ChatChat 聊聊APP

> 单机优先、零自建服务器成本、隐私本地化的 AI 记忆助手与理性咨询 App。
> 让用户把人生经历沉淀为本地可检索的长期语义记忆，并在未来对话中被 AI 安全调用。

---

## 1. 产品概述

| 项 | 详情 |
|---|---|
| **类型** | React Native (Expo) 移动端 App |
| **平台** | iOS / Android 原生（Development Build） |
| **代码仓库** | `C:\Users\LuzekkA\Desktop\ChatChat` |

Chat Chat 不是普通聊天机器人，也不是单纯日记工具。核心价值在于将用户的对话、日记、人生事件与个人画像沉淀为**本地长期记忆系统**，通过 RAG 在每次对话中自动检索相关记忆并注入 Prompt，实现个性化 AI 陪伴。

---

## 2. 核心原则

| 原则 | 含义 |
|---|---|
| **零自建服务器成本** | 用户自带 OpenAI 兼容 API Key，无后端服务 |
| **本地隐私优先** | 记忆、向量索引全部本地 SQLite，不上传 |
| **前台/后台解耦** | 聊天不等待记忆抽取，fire-and-forget 后台处理 |
| **渐进降级** | 无网络/无 API Key 时仍可离线记录日记 |

---

## 3. 核心功能

- **AI 人生对话** — 流式聊天，支持 OpenAI/Anthropic 双通道，对话自动触发记忆沉淀
- **长期记忆系统** — Vector（BGE 512-dim 语义检索）+ FTS5（BM25 关键词）+ Graph（BFS 因果图遍历）三路混合检索，RRF 融合排序
- **人生时间线** — 从对话中自动提取里程碑事件，向量去重
- **自传生成** — Beat Loop 架构，5 层 Prompt 流水线，支持作家风格模仿（余华、鲁迅等）
- **日记模块** — 热力图日历 + 情绪评分，离线可用
- **记忆探索器** — SVG 图谱可视化，多维度浏览记忆/事件/画像
- **用户引导系统** — 三层分层引导（Welcome + Spotlight Tour + 交互引导）

---

## 4. 技术特色

| 层级 | 选型 |
|---|---|
| 框架 | React Native 0.81 + Expo SDK 54 |
| 数据库 | expo-sqlite + sqlite-vec 向量扩展 + FTS5 全文索引 |
| 本地 Embedding | onnxruntime-react-native (Native JSI) + BGE-small-zh-v1.5 (512-dim) |
| 状态管理 | Zustand 5 |
| 外部 LLM | OpenAI 兼容 / Anthropic 原生 API（适配器模式双通道） |
| 路由 | Expo Router 6 (file-based) |
| 测试 | Jest + jest-expo（315+ tests） |

---

## 5. 工程架构与文档体系

### 5.1 文档治理

项目采用严格的文档分类和权威来源制度，每种知识只有一个权威文档，其他文档只能引用。

```
docs/
├── requirements/     ← 产品需求（单文件）
├── architecture/     ← 技术架构 + 各模块设计（SPEC、记忆、自传、引导、探索器）
├── api/              ← API 契约
├── database/         ← 数据库迁移方案
├── development/      ← 开发任务拆解
├── testing/          ← 测试文档与运行指南
├── reference/        ← TypeScript 类型契约
├── history/          ← 实施路线图 + 开发日志
├── AI_CONTEXT.md     ← AI Agent 最小上下文规则（任务×文档映射表，减少 Token 消耗）
├── SOURCE_OF_TRUTH.md ← 知识权威来源映射表
└── PRIVACY_POLICY.md  ← 隐私政策
```

### 5.2 AI Agent 协作机制

项目维护了完整的 AI Agent 协作体系：

- **`AI_CONTEXT.md`** — 定义 10 种任务类型各自的最小必读文档集，将 Token 消耗从全量 87,500 减少 70-90%
- **`SOURCE_OF_TRUTH.md`** — 规定每种知识域的权威文档，防止多文档重复维护
- **`MEMORY.md`**（项目根目录）— AI Agent 行为约束、变更分级（Tier 1-3）、敏感文件清单
- **`.claudecode/templates/`** — Claude Code 交互模板

核心原则：**AI Agent 每次只加载完成任务所需的最小文档，禁止全量加载。**

### 5.3 变更分级制度

| Tier | 说明 | 要求 |
|---|---|---|
| Tier 1 | 简单修复（typo、样式、小重构） | 仅运行测试 |
| Tier 2 | 功能变更（新增模块、API 改动） | 同步更新对应文档 + 测试 |
| Tier 3 | 核心风险区（chatStore、数据库迁移、安全相关） | 需确认后方可编辑，同步更新所有关联文档 |

---

> 最后更新：2026-08-09
