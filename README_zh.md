[中文版](./README_zh.md) | [English](./README.md)

# lark-workflow

> 15 个基于[飞书 CLI](https://github.com/larksuite/cli) 的跨域工作流 Skills — 将多个业务域编排为强大的自动化工作流。

[飞书 CLI](https://github.com/larksuite/cli) 提供了 22 个原子 Skills 覆盖各个独立业务域（日历、消息、文档、任务等），**lark-workflow** 则将它们组合为多步骤、跨域的工作流，解决真实的效率场景。

## 快速开始

### 1. 安装

**方式 A：通过 npx（GitHub 源）**

```bash
# 安装飞书 CLI（如未安装）
npm install -g @larksuite/cli

# 安装飞书 CLI 官方 Skills（必需依赖）
npx skills add larksuite/cli -y -g

# 安装 lark-workflow Skills
npx skills add woodfantasy/lark-workflow -y -g
```

**方式 B：通过 OpenClaw（ClawHub）**

```bash
openclaw install lark-workflow
```

### 2. 认证

```bash
# 推荐：一次性授权所有常用域
lark-cli auth login --recommend
```

### 3. 使用

用自然语言告诉 AI Agent 你的需求即可：

```
💬 "今天有什么安排"                → daily-briefing（每日简报）
💬 "帮我写周报"                   → weekly-report（周报生成）
💬 "把会议纪要变成任务"            → action-extractor（任务提取）
💬 "帮我总结这个文档"              → doc-summarizer（文档摘要）
💬 "知识库里关于 XX 怎么说的"       → knowledge-qa（知识问答）
💬 "今天应该先做什么"              → task-prioritizer（任务排序）
💬 "帮我约张三李四下周开会"         → smart-scheduler（智能排会）
💬 "我的审批到哪了"                → approval-accelerator（审批催办）
💬 "分析一下这个多维表格"           → base-analytics（数据分析）
💬 "新同事小王入职了"              → onboarding（入职自动化）
💬 "用模板批量生成文档"            → doc-template-engine（模板引擎）
💬 "这周开了多少会"               → meeting-efficiency（会议效率）
💬 "谁比较空可以接新任务"           → workload-balancer（负载均衡）
💬 "知识库哪些文档过时了"           → wiki-auditor（知识审计）
💬 "帮我优化明天的日程"            → calendar-optimizer（日程优化）
```

AI Agent 会自动读取对应的 SKILL.md，理解工作流，执行正确的 lark-cli 命令。

## Skills 一览

### 🔄 个人效率

| Skill | 说明 | 涉及业务域 |
|-------|------|-----------|
| [`daily-briefing`](./skills/lark-workflow-daily-briefing/SKILL.md) | 每日全景简报 — 聚合 5 个域的信息 | 日历、任务、邮箱、审批、即时通讯 |
| [`weekly-report`](./skills/lark-workflow-weekly-report/SKILL.md) | 周报自动生成 — 汇总任务、会议、文档活动 | 日历、任务、云空间、视频会议、云文档 |
| [`task-prioritizer`](./skills/lark-workflow-task-prioritizer/SKILL.md) | 智能任务排序 — 紧急/重要矩阵 + 时间块建议 | 任务、日历 |
| [`calendar-optimizer`](./skills/lark-workflow-calendar-optimizer/SKILL.md) | 日程优化 — 冲突诊断、碎片分析、重排建议 | 日历 |

### 🤝 团队协作

| Skill | 说明 | 涉及业务域 |
|-------|------|-----------|
| [`smart-scheduler`](./skills/lark-workflow-smart-scheduler/SKILL.md) | 智能排会 — 多人忙闲查询 + 会议室预约 + 一键创建日程 | 日历、通讯录 |
| [`action-extractor`](./skills/lark-workflow-action-extractor/SKILL.md) | 会议任务闭环 — 纪要→行动项→任务→通知 | 视频会议、妙记、云文档、通讯录、任务、即时通讯 |
| [`onboarding`](./skills/lark-workflow-onboarding/SKILL.md) | 新人入职自动化 — 编排 5 个域 | 通讯录、即时通讯、任务、知识库、日历 |
| [`workload-balancer`](./skills/lark-workflow-workload-balancer/SKILL.md) | 团队负载均衡 — 负载热力图 + 分配建议 | 任务、日历、通讯录 |
| [`approval-accelerator`](./skills/lark-workflow-approval-accelerator/SKILL.md) | 审批加速器 — 进度追踪 + 超时催办 | 审批、即时通讯、通讯录 |

### 📊 数据与分析

| Skill | 说明 | 涉及业务域 |
|-------|------|-----------|
| [`base-analytics`](./skills/lark-workflow-base-analytics/SKILL.md) | 多维表格智能分析 — 趋势、异常、分布洞察 | 多维表格 |
| [`meeting-efficiency`](./skills/lark-workflow-meeting-efficiency/SKILL.md) | 会议效率分析 — 时间成本、纪要覆盖率、节奏 | 视频会议、日历、妙记 |
| [`doc-template-engine`](./skills/lark-workflow-doc-template-engine/SKILL.md) | 文档模板引擎 — 数据驱动批量生成文档 | 云文档、多维表格、电子表格 |

### 📚 知识管理

| Skill | 说明 | 涉及业务域 |
|-------|------|-----------|
| [`doc-summarizer`](./skills/lark-workflow-doc-summarizer/SKILL.md) | 文档智能摘要 — 结构化提取要点和决策 | 云文档、知识库、云空间 |
| [`knowledge-qa`](./skills/lark-workflow-knowledge-qa/SKILL.md) | 知识库问答 — RAG 式问答 + 来源引用 | 知识库、云文档、云空间 |
| [`wiki-auditor`](./skills/lark-workflow-wiki-auditor/SKILL.md) | 知识库健康审计 — 过时检测 + 健康度评分 | 知识库、云空间 |

## 工作原理

每个 Skill 都是一个 **SKILL.md 工作流定义文件**，它指导 AI Agent 如何跨多个域编排 lark-cli 命令，以结构化的方式逐步完成复杂任务。

```
┌─────────────────────────────────────────────────────────────────────┐
│                          AI Agent                                    │
│     你说："今天有什么安排"                                            │
│     Agent 读取：daily-briefing/SKILL.md                             │
│     Agent 执行：calendar → task → mail → approval → AI 汇总简报      │
└──────────┬──────────────────────────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────────────────────────────┐
│                   lark-workflow Skills（15 个）                       │
│                                                                      │
│  个人效率          团队协作          数据分析        知识管理          │
│  ┌────────────┐  ┌──────────────┐  ┌────────────┐  ┌────────────┐  │
│  │每日简报    │  │智能排会      │  │表格分析    │  │文档摘要    │  │
│  │周报生成    │  │会议任务闭环  │  │会议效率    │  │知识问答    │  │
│  │任务排序    │  │入职自动化    │  │模板引擎    │  │知识审计    │  │
│  │日程优化    │  │负载均衡      │  └────────────┘  └────────────┘  │
│  └────────────┘  │审批加速      │                                    │
│                  └──────────────┘                                    │
└──────────┬──────────────────────────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────────────────────────────┐
│                  飞书 CLI 原子 Skills（22 个）                        │
│  日历 · 任务 · 邮箱 · 审批 · 消息 · 会议 · 文档 · 云空间            │
│  通讯录 · 妙记 · 多维表格 · 电子表格 · 知识库 · 事件 · 画板         │
│  演示文稿 · 考勤 · API 浏览器 · Skill 创建器 · 共享 · 画板CLI       │
└─────────────────────────────────────────────────────────────────────┘
```

### SKILL.md 包含什么？

每个 Skill 遵循 [lark-skill-maker](https://github.com/larksuite/cli/tree/main/skills/lark-skill-maker) 标准：

- **YAML frontmatter** — 名称、版本、描述、依赖声明
- **触发场景** — 自然语言触发词列表
- **工作流步骤** — Step 1 → 2 → 3... 每步附带精确的 CLI 命令
- **输出模板** — 结构化 Markdown 格式，确保输出一致
- **权限表** — 每个命令所需的 OAuth scope
- **容错机制** — 某个域不可用时的优雅降级

## 认证速查表

每个 Skill 在其 SKILL.md 中列出了所需权限。以下是快速参考：

| Skill | 认证命令 |
|-------|---------|
| daily-briefing | `lark-cli auth login --domain calendar,task,mail,approval,im` |
| weekly-report | `lark-cli auth login --domain calendar,task,drive,vc,doc` |
| action-extractor | `lark-cli auth login --domain vc,drive,contact,task,im` |
| doc-summarizer | `lark-cli auth login --domain doc,drive,wiki` |
| knowledge-qa | `lark-cli auth login --domain wiki,doc,drive` |
| task-prioritizer | `lark-cli auth login --domain task,calendar` |
| smart-scheduler | `lark-cli auth login --domain calendar,contact` |
| approval-accelerator | `lark-cli auth login --domain approval,im,contact` |
| base-analytics | `lark-cli auth login --domain base,doc` |
| onboarding | `lark-cli auth login --domain contact,im,task,wiki,calendar` |
| doc-template-engine | `lark-cli auth login --domain doc,base,drive` |
| meeting-efficiency | `lark-cli auth login --domain vc,calendar` |
| workload-balancer | `lark-cli auth login --domain task,calendar,contact` |
| wiki-auditor | `lark-cli auth login --domain wiki,drive` |
| calendar-optimizer | `lark-cli auth login --domain calendar` |

> **提示**：使用 `lark-cli auth login --recommend` 一次性授权最常用的 scope。

> **最小权限方式**：如果你只使用部分 Skill，可以按上表中每行的命令仅授权所需的域，避免过度授权。

## 安全与隐私

- **不存储密钥** — lark-workflow 是纯指令文件（SKILL.md），不读取、存储或传输任何凭证。认证由 `lark-cli auth` 处理，Token 存储在系统钥匙串中。
- **不执行代码** — 安装仅添加 Markdown 文件，不安装脚本、二进制文件或后台进程。
- **需用户确认** — 所有写操作（创建日程、发送消息、创建任务等）执行前必须经用户确认。
- **权限透明** — 每个 Skill 在权限表中声明所需的 OAuth scope，支持按需最小授权。
- **优雅降级** — 可选域未授权或执行失败时，仅在输出中标注提示，不阻塞主流程。

## 贡献

欢迎社区贡献！请阅读 [CONTRIBUTING.md](./CONTRIBUTING.md) 了解贡献规范。

## 许可证

本项目基于 **MIT 许可证** 开源 — 详见 [LICENSE](./LICENSE)。

本软件运行时会调用[飞书开放平台 API](https://open.feishu.cn/)，使用这些 API 需遵守[飞书用户服务协议](https://www.feishu.cn/terms)和[飞书隐私政策](https://www.feishu.cn/privacy)。
