# VeriFact

### 基于多模型 Agent 与人工复核的可解释多模态事实核查平台

[![Portfolio](https://img.shields.io/badge/type-engineering%20portfolio-1f6feb)](#)
[![UNSW COMP9900](https://img.shields.io/badge/UNSW-COMP9900-ffd100)](#)
[![Result](https://img.shields.io/badge/课程成绩-87%2F100%20HD-2ea44f)](#)

**作品集作者：[李坤宇 / Kunyu Li（@like-rich）](https://github.com/like-rich)** · [English Version](README_EN.md)

VeriFact 是一个支持文本、图片、音频、视频、网页 URL 与在线视频链接的事实核查平台。系统将复杂输入拆解为可独立验证的子主张，检索网络证据，调用多个 LLM 并行评估，再通过能力加权共识生成包含结论、置信度、推理过程和来源链接的可审计报告。

当结果置信度较低、证据互相冲突或被质量抽检时，任务会进入 Reviewer 队列，由人工提交 Ground Truth，形成持续评估 AI 表现的反馈闭环。

> 本仓库是公开的**项目作品集与架构案例**。原始实现保存在 UNSW 私有团队仓库中，本仓库不复制课程源码。

## 项目展示

### 系统架构

![VeriFact 系统架构图](assets/system-architecture.png)

系统采用 React/Vite 前端与 FastAPI 后端，后端按 Router、Service、Repository、Model 分层；PostgreSQL 保存用户、核查结果、人工审核、AI 使用量和审计记录，外部集成包括 LLM、证据检索、邮件服务及本地媒体工具。

### AI Agent 工作流

![VeriFact AI 核查工作流](assets/verification-workflow.png)

工作流遵循“先证据、后判断”的原则。缺少可靠证据或模型分歧过大时，系统返回 `Uncertain`，而不是输出缺乏依据的确定结论。

> 产品界面截图和 Demo GIF 将在确认课程与团队材料授权后补充。本仓库不会用虚构界面冒充真实成果。

## 项目概览

| | |
| --- | --- |
| **项目背景** | UNSW COMP9900 Capstone 团队项目 |
| **课程成绩** | 87/100，High Distinction |
| **我的重点** | AI 核查流水线、证据质量、多模型共识、Reviewer 工作流与评估看板 |
| **后端** | Python、FastAPI、Pydantic、异步 SQLAlchemy、Alembic |
| **数据库** | PostgreSQL、JSONB、版本化数据库迁移 |
| **前端** | React、Vite、React Router、Axios、Recharts |
| **AI 与检索** | LangChain、多家 OpenAI-compatible 模型、Tavily / DuckDuckGo |
| **工程交付** | Docker Compose、GitHub Actions、pytest、ESLint、Ruff |

## 核心功能

- **多模态输入**：支持文本、图片、音频、视频、URL 与在线视频。
- **主张拆解**：将复杂输入拆成 1–5 个独立、可检索的事实点。
- **证据检索**：并行生成查询，检索网页证据并评估来源质量。
- **多模型验证**：多个 LLM 对同一主张和证据分别给出结构化判断。
- **能力加权共识**：保留模型分歧，基于权重、置信度和 supermajority 规则聚合结果。
- **实时进度**：通过 Server-Sent Events 推送流水线阶段和部分结果。
- **人工复核**：低置信度、`Uncertain`、来源冲突及随机抽检结果进入 Reviewer 队列。
- **评估分析**：提供一致率、错误置信率、混淆矩阵、校准曲线和错误类型分布。
- **多角色管理**：支持普通用户、Reviewer 与 Administrator 的不同工作界面。
- **Docker 部署**：使用 Docker Compose 编排前端、后端和 PostgreSQL。

## 技术架构

```text
React / Vite SPA
       │ REST · multipart · SSE
       ▼
FastAPI Gateway + Middleware
       │
       ├── Auth Service
       ├── Verification Service ── AI Agent / Media Pipeline
       ├── Review Service ──────── Smart Queue / Analytics
       └── Admin Service
                  │
       Repository Layer + SQLAlchemy
                  │
             PostgreSQL 16
```

后端分层设计将 HTTP、业务规则、数据访问和持久化解耦。AI 服务、搜索服务与媒体工具通过独立边界接入，使外部提供方发生故障时可以降级处理，而不会在核心业务层制造虚假成功结果。

详细说明：[系统架构](docs/architecture.md) · [后端设计](docs/backend-design.md)

## AI Agent 工作流

1. **输入标准化**：抽取文本、语音转写、关键帧、元数据和来源信息。
2. **主张拆解**：生成 1–5 个自包含、可搜索的子主张。
3. **并行证据检索**：为每个子主张生成多组查询，并对不足结果扩大检索范围。
4. **来源质量标注**：在模型评估前标注可信度、时效性和相关性。
5. **多模型独立判断**：要求各模型返回可验证的结构化字段，而不是自由文本。
6. **加权共识**：结合模型能力权重、置信度、模型一致性和证据重合度。
7. **全局结果聚合**：将子主张结果汇总为 `Supported`、`Refuted` 或 `Uncertain`。
8. **人工反馈闭环**：将高风险结果送入 Reviewer 队列并生成质量分析指标。

详细说明：[AI 工作流与共识设计](docs/ai-workflow.md)

## 核心技术实现

### 证据优先与可信来源处理

模型不能只依靠参数记忆回答。每次判断都绑定检索证据，最终来源 URL 必须来自已获取的材料。没有可用来源时，系统明确降低结论强度并返回风险提示。

### 多模型并行与加权共识

不同模型使用相同子主张和证据独立作答。系统保留模型级结果，通过能力权重和加权三分之二多数规则生成共识；当模型不一致时对最终置信度施加惩罚。

### SSE 流式反馈

核查过程包含搜索与模型调用，延迟具有不确定性。SSE 会推送 `stage_start`、`stage_done`、`partial` 和 `complete` 等事件，让用户在最终结果产生前看到实时进度。

### Human-in-the-loop 质量控制

Reviewer 队列同时覆盖风险驱动与随机抽检：置信度低于 70%、不确定结论、证据冲突，以及 15% 的高置信度结果抽样。随机抽检能够发现“高置信度但错误”的情况，避免评估数据只覆盖模型主动承认不确定的样本。

### 数据与迁移

PostgreSQL 保存结构化业务数据，JSONB 保存不断演化的推理链和媒体报告；Alembic 管理可追踪的数据库版本。审核任务与人工判定分表保存，便于独立演化队列策略和 Ground Truth 数据。

## 我的贡献

以下内容根据私有仓库中与 `like-rich` / 李坤宇对应的分支、提交和文件差异整理。

### AI 事实核查后端

- 实现核心 Agent 编排，将输入拆分为独立子主张。
- 实现并行多查询检索、URL 证据处理与失败降级。
- 定义结论、置信度、证据摘要、推理步骤、来源和免责声明等结构化输出。
- 完成从模型级结果到子主张共识、再到全局结论的聚合逻辑。

### 可信来源与多模型验证

- 开发来源可信度分类与证据标注逻辑。
- 对接多个兼容模型提供方并进行并行验证。
- 实现能力加权投票、supermajority、分歧惩罚与不确定性处理。

### Reviewer 工作流与评估看板

- 贯通数据库模型、Alembic 迁移、Repository、Service、FastAPI 路由和 React 页面。
- 实现智能入队、Ground Truth、审核历史、错误分类和重复审核保护。
- 实现 AI–人工一致率、错误置信率、混淆矩阵、校准数据和错误分布统计。
- 将审核入队逻辑接入普通核查与 SSE 流式核查路径。

### 集成与项目文档

- 完成 Reviewer 页面、API、导航与报告流程的集成和调整。
- 编写项目根文档，覆盖架构、配置、本地运行与 Docker 部署。

更完整的职责边界和提交依据：[我的贡献](docs/my-contributions.md)

## 团队贡献边界

VeriFact 是团队项目。认证系统、管理员模块、媒体处理流水线、部分前端基础能力和后续安全加固包含其他成员的重要贡献。本作品集会介绍这些团队级能力以说明最终系统，但不会把它们表述为我独立完成的工作。

## 项目成果

- 完成端到端的多模态事实核查原型。
- 将多模型验证、证据质量与人工复核组合为完整闭环。
- 通过 Docker Compose 提供可复现的三层运行环境。
- 完成 REST API、SSE、数据库迁移、审核分析与前后端集成。
- UNSW COMP9900 课程成绩：**87/100，High Distinction**。

## 文档导航

| 文档 | 内容 |
| --- | --- |
| [系统架构](docs/architecture.md) | 组件关系、数据流、部署与设计取舍 |
| [AI 工作流](docs/ai-workflow.md) | 主张拆解、检索、多模型验证和共识算法 |
| [后端设计](docs/backend-design.md) | 分层结构、异步边界、可靠性和安全设计 |
| [数据库设计](docs/database-design.md) | 概念 ER 模型、JSONB、队列与审核记录 |
| [API 设计](docs/api-design.md) | REST、SSE、认证、错误与 Reviewer 接口 |
| [我的贡献](docs/my-contributions.md) | 个人职责、实现范围与团队边界 |

## 源码与 Academic Integrity 说明

VeriFact 是 **UNSW COMP9900 团队课程项目**。为遵守课程规定、团队成员权益和 Academic Integrity 要求：

- 本公开仓库仅包含作品集文档、架构图和高层设计；
- 不公开私有课程源码、密钥、数据集或团队 Git 历史；
- 原始实现继续保存在 UNSW 私有团队仓库；
- 是否公开部分个人代码，将在确认学校政策和团队授权后决定。

本仓库暂不添加开源许可证。没有许可证即不表示相关材料可以被自由复制、修改或再分发。

## 作者

**李坤宇 / Kunyu Li** · [GitHub @like-rich](https://github.com/like-rich)

如果你正在评估我的后端或 AI 应用开发能力，建议优先阅读：[我的贡献](docs/my-contributions.md)、[AI 工作流](docs/ai-workflow.md) 和 [后端设计](docs/backend-design.md)。
