[中文版](./README_zh.md) | [English](./README.md)

# lark-workflow

基于 [飞书 CLI](https://github.com/larksuite/cli) 的跨域工作流 Skills 集合 — 将多个业务域编排为强大的自动化工作流。

[飞书 CLI](https://github.com/larksuite/cli) 提供了 18 个原子 Skills 覆盖各个独立业务域（日历、消息、文档、任务等），**lark-workflow** 则将它们组合为多步骤、跨域的工作流，解决真实的效率场景。

## Skills 一览

| Skill | 说明 | 涉及业务域 |
|-------|------|-----------|
| [`lark-workflow-daily-briefing`](./skills/lark-workflow-daily-briefing/SKILL.md) | 每日全景简报 — 聚合日程、任务、邮件、审批、消息，一站式生成早间简报 | 日历、任务、邮箱、审批、即时通讯 |
| [`lark-workflow-weekly-report`](./skills/lark-workflow-weekly-report/SKILL.md) | 周报自动生成 — 汇总已完成任务、会议纪要、文档活动，生成结构化周报 | 日历、任务、云空间、视频会议、云文档 |
| [`lark-workflow-action-extractor`](./skills/lark-workflow-action-extractor/SKILL.md) | 会议任务闭环 — 从会议纪要提取行动项，自动创建任务并通知责任人 | 视频会议、妙记、云文档、通讯录、任务、即时通讯 |
| [`lark-workflow-doc-summarizer`](./skills/lark-workflow-doc-summarizer/SKILL.md) | 文档智能摘要 — 读取飞书文档或知识库文档，生成结构化摘要（要点、决策、待确认项） | 云文档、知识库、云空间 |
| [`lark-workflow-knowledge-qa`](./skills/lark-workflow-knowledge-qa/SKILL.md) | 知识库问答 — 基于知识库的 RAG 式问答，返回精准回答并附上来源引用 | 知识库、云文档、云空间 |
| [`lark-workflow-task-prioritizer`](./skills/lark-workflow-task-prioritizer/SKILL.md) | 智能任务排序 — 基于紧急/重要矩阵分析，结合日程约束生成时间块建议 | 任务、日历 |
| [`lark-workflow-smart-scheduler`](./skills/lark-workflow-smart-scheduler/SKILL.md) | 智能排会 — 查询多人忙闲状态，找出共同空闲时段并一键创建日程 | 日历、通讯录 |
| [`lark-workflow-approval-accelerator`](./skills/lark-workflow-approval-accelerator/SKILL.md) | 审批加速器 — 追踪审批进度，检测超时实例，发送催办消息 | 审批、即时通讯、通讯录 |

## 安装

### 前置条件

- 已安装并完成认证的 [飞书 CLI](https://github.com/larksuite/cli)
- Node.js（npm/npx）

```bash
# 安装飞书 CLI（如未安装）
npm install -g @larksuite/cli

# 安装飞书 CLI 官方 Skills（必需）
npx skills add larksuite/cli -y -g

# 安装 lark-workflow Skills
npx skills add woodfantasy/lark-workflow -y -g
```

### 认证授权

每个 Skill 需要特定的业务域权限，使用前请先授权：

```bash
# daily-briefing（5 个域）
lark-cli auth login --domain calendar,task,mail,approval,im

# weekly-report
lark-cli auth login --domain calendar,task,drive,vc,doc

# action-extractor
lark-cli auth login --domain vc,drive,contact,task,im

# 或使用 --recommend 自动选择常用权限
lark-cli auth login --recommend
```

## 工作原理

这些 Skills 是**基于 SKILL.md 的工作流定义** — 它们指导 AI Agent 如何跨多个域编排 lark-cli 命令，以结构化的方式逐步完成复杂任务。

```
┌───────────────────────────────────────────────────────┐
│                    AI Agent                            │
│  读取 SKILL.md → 理解工作流 → 执行 CLI 命令            │
└─────────┬─────────────────────────────────────────────┘
          │
          ▼
┌───────────────────────────────────────────────────────┐
│              lark-workflow Skills                       │
│  ┌──────────┐  ┌──────────┐  ┌──────────────────┐    │
│  │  每日简报 │  │  周报生成 │  │   会议任务闭环    │    │
│  └────┬─────┘  └────┬─────┘  └────┬─────────────┘    │
└───────┼──────────────┼─────────────┼──────────────────┘
        │              │             │
        ▼              ▼             ▼
┌───────────────────────────────────────────────────────┐
│              飞书 CLI 原子 Skills                       │
│  日历 · 任务 · 邮箱 · 审批 · 消息 · 会议 · 文档       │
│  云空间 · 通讯录 · 妙记 · 多维表格 · 电子表格 · 知识库  │
└───────────────────────────────────────────────────────┘
```

## 贡献

欢迎社区贡献！请阅读 [CONTRIBUTING.md](./CONTRIBUTING.md) 了解贡献规范。

## 许可证

本项目基于 **MIT 许可证** 开源 — 详见 [LICENSE](./LICENSE)。

本软件运行时会调用 [飞书开放平台 API](https://open.feishu.cn/)，使用这些 API 需遵守[飞书用户服务协议](https://www.feishu.cn/terms)和[飞书隐私政策](https://www.feishu.cn/privacy)。
