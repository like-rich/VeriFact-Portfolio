# VeriFact

### 基于多模型 Agent 与人工复核的可解释多模态事实核查平台

[![Portfolio](https://img.shields.io/badge/type-engineering%20portfolio-1f6feb)](#)
[![UNSW COMP9900](https://img.shields.io/badge/UNSW-COMP9900-ffd100)](#)
[![Course Result](https://img.shields.io/badge/course%20result-87%2F100%20HD-2ea44f)](#)

**作者：[李坤宇 / Kunyu Li（@like-rich）](https://github.com/like-rich)** · [English Version](README_EN.md)

VeriFact 支持文本、图片、音频、视频和 URL 事实核查。系统先拆解可验证子主张并检索网络证据，再由多个 LLM 独立评估，通过能力加权共识生成包含结论、置信度、推理过程和来源链接的报告；高风险结果会进入人工 Reviewer 队列。

> 本仓库是公开作品集。原始实现保存在 UNSW 私有团队仓库中，此处不复制课程源码。

## 项目概览与技术栈

| | |
| --- | --- |
| **项目背景** | UNSW COMP9900 Capstone 团队项目 |
| **后端** | Python、FastAPI、Pydantic、异步 SQLAlchemy、Alembic |
| **前端** | React、Vite、Axios、Recharts |
| **数据与 AI** | PostgreSQL、LangChain、多家 OpenAI-compatible 模型 |
| **检索与媒体** | Tavily / DuckDuckGo、FFmpeg、Whisper、ExifTool、yt-dlp |
| **交付与质量** | Docker Compose、GitHub Actions、pytest、Vitest、ESLint、Ruff |

## 产品展示

真实界面截图与 Demo GIF 尚未公开，原因是仍需确认课程及团队对 UI 素材的公开授权。确认后计划补充首页、核查报告、Reviewer 队列和 Analytics 看板，并清除账户及用户数据。本仓库不会用示意图冒充真实运行截图。

## My Contributions

- **AI 事实核查后端：** 实现主张拆解、并行证据检索、结构化模型评估和全局结果聚合。
- **多模型共识：** 开发可信来源标注、能力加权投票、supermajority、分歧惩罚与不确定性处理。
- **Reviewer 工作流：** 贯通数据库迁移、Repository、Service、FastAPI、React 队列及评估看板。
- **系统集成：** 将智能入队接入普通与 SSE 流式核查，并完成 Reviewer 页面、API 和报告流程联调。

公开的技术证据：[个人贡献](docs/my-contributions.md) · [共识算法伪代码](docs/consensus-algorithm.md) · [Reviewer 智能队列](docs/reviewer-queue.md)

VeriFact 是团队项目。认证、管理员、媒体流水线、部分前端基础与后续安全加固包含其他成员的重要贡献，不在此表述为我的独立成果。

## 系统架构

![VeriFact 系统架构图](assets/system-architecture.png)

React/Vite 通过 REST、multipart 和 SSE 访问 FastAPI；后端采用 Router → Service → Repository → Model 分层，PostgreSQL 保存核查、审核、AI 使用量与审计数据。

[查看完整系统架构](docs/architecture.md) · [查看后端设计](docs/backend-design.md)

## 核心技术亮点

| 方向 | 实现摘要 |
| --- | --- |
| **多模型事实核查** | LangChain 编排事实拆解、并行检索、结构化验证及能力加权共识 |
| **Evidence-first** | 来源质量标注；没有可用证据时返回 `Uncertain`，不生成虚假引用 |
| **流式处理** | SSE 推送 `stage_start`、`stage_done`、`partial` 和 `complete` |
| **Human-in-the-loop** | 低置信度、冲突及 15% 高置信度抽样进入 Reviewer 队列 |
| **后端工程** | 分层服务、异步数据库、版本迁移、事务顺序、缓存和 Docker 部署 |

[查看完整 AI Workflow 设计](docs/ai-workflow.md)

## 项目成果

- 完成端到端多模态事实核查与人工质量反馈闭环。
- 自动化测试：**后端 18 项 + 前端 17 项，共 35 项通过**；[查看测试范围与限制](docs/test-results.md)。
- **UNSW COMP9900 Course Result: 87/100 (High Distinction)**。

这里展示的是个人课程最终成绩，不将其描述为团队项目的独立评分。

## 技术文档

| 文档 | 主要内容 |
| --- | --- |
| [AI 工作流](docs/ai-workflow.md) | 拆解、检索、验证、聚合与失败处理（英文） |
| [共识算法](docs/consensus-algorithm.md) | 伪代码、示例、复杂度与方案对比（英文） |
| [Reviewer 智能队列](docs/reviewer-queue.md) | 入队规则、状态、事务与生产改进（英文） |
| [API 设计审查](docs/api-design.md) | 已验证接口与设计改进分离说明（英文） |
| [后端与数据库](docs/backend-design.md) | 分层、并发、可靠性、安全和数据模型（英文） |
| [个人贡献](docs/my-contributions.md) | 个人职责证据与团队边界（英文） |

## 源码与授权说明

为遵守课程规定、团队权益和 Academic Integrity 要求，本仓库仅公开作品集文档与架构图，不公开私有源码、密钥、数据集或团队 Git 历史。是否公开个人代码或真实界面素材，将在确认学校政策和团队授权后决定。

本仓库暂不添加开源许可证；没有许可证不代表允许复制、修改或再分发。

**李坤宇 / Kunyu Li** · [GitHub @like-rich](https://github.com/like-rich)
