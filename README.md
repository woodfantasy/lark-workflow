[中文版](./README_zh.md) | [English](./README.md)

# lark-workflow

Cross-domain workflow skills for [Lark/Feishu CLI](https://github.com/larksuite/cli) — orchestrate multiple business domains into powerful automated workflows.

While [lark-cli](https://github.com/larksuite/cli) provides 18 atomic skills for individual domains (Calendar, IM, Docs, Tasks, etc.), **lark-workflow** combines them into multi-step, cross-domain workflows that solve real-world productivity scenarios.

## Skills

| Skill | Description | Domains Used |
|-------|-------------|-------------|
| [`lark-workflow-daily-briefing`](./skills/lark-workflow-daily-briefing/SKILL.md) | Daily panoramic briefing — aggregates calendar, tasks, mail, approvals, and messages into a single morning report | Calendar, Task, Mail, Approval, IM |
| [`lark-workflow-weekly-report`](./skills/lark-workflow-weekly-report/SKILL.md) | Automated weekly report — collects completed tasks, meetings, and document activity to generate a structured weekly summary | Calendar, Task, Drive, VC, Docs |
| [`lark-workflow-action-extractor`](./skills/lark-workflow-action-extractor/SKILL.md) | Meeting-to-task closed loop — extracts action items from meeting notes and auto-creates tasks with assignee notifications | VC, Minutes, Docs, Contact, Task, IM |
| [`lark-workflow-doc-summarizer`](./skills/lark-workflow-doc-summarizer/SKILL.md) | Document summarizer — reads Lark Docs or Wiki pages and generates structured summaries with key points and decisions | Docs, Wiki, Drive |
| [`lark-workflow-knowledge-qa`](./skills/lark-workflow-knowledge-qa/SKILL.md) | Knowledge base Q&A — RAG-style question answering over Lark Wiki with source citations | Wiki, Docs, Drive |
| [`lark-workflow-task-prioritizer`](./skills/lark-workflow-task-prioritizer/SKILL.md) | Task prioritizer — Eisenhower matrix analysis with time-block suggestions based on calendar constraints | Task, Calendar |
| [`lark-workflow-smart-scheduler`](./skills/lark-workflow-smart-scheduler/SKILL.md) | Smart scheduler — finds common free time slots across attendees and creates calendar events | Calendar, Contact |
| [`lark-workflow-approval-accelerator`](./skills/lark-workflow-approval-accelerator/SKILL.md) | Approval accelerator — tracks approval status, detects timeouts, and sends nudge messages | Approval, IM, Contact |
| [`lark-workflow-base-analytics`](./skills/lark-workflow-base-analytics/SKILL.md) | Bitable analytics — trend analysis, anomaly detection, and distribution insights for multidimensional tables | Base |
| [`lark-workflow-onboarding`](./skills/lark-workflow-onboarding/SKILL.md) | New hire onboarding — auto-joins chats, creates task checklists, sends wiki docs, and schedules training | Contact, IM, Task, Wiki, Calendar |
| [`lark-workflow-doc-template-engine`](./skills/lark-workflow-doc-template-engine/SKILL.md) | Document template engine — fills Docs templates with Base/Sheets data for batch document generation | Docs, Base, Sheets |
| [`lark-workflow-meeting-efficiency`](./skills/lark-workflow-meeting-efficiency/SKILL.md) | Meeting efficiency analysis — measures meeting time cost, note coverage, and schedule patterns | VC, Calendar, Minutes |
| [`lark-workflow-workload-balancer`](./skills/lark-workflow-workload-balancer/SKILL.md) | Team workload balancer — visualizes team member load and suggests optimal task assignments | Task, Calendar, Contact |
| [`lark-workflow-wiki-auditor`](./skills/lark-workflow-wiki-auditor/SKILL.md) | Wiki health auditor — detects outdated docs, structural issues, and generates health scores | Wiki, Drive |
| [`lark-workflow-calendar-optimizer`](./skills/lark-workflow-calendar-optimizer/SKILL.md) | Calendar optimizer — diagnoses conflicts, fragmentation, and suggests schedule improvements | Calendar |

## Installation

### Prerequisites

- [lark-cli](https://github.com/larksuite/cli) installed and authenticated
- Node.js (npm/npx)

```bash
# Install lark-cli (if not already installed)
npm install -g @larksuite/cli

# Install lark-cli official skills (required)
npx skills add larksuite/cli -y -g

# Install lark-workflow skills
npx skills add woodfantasy/lark-workflow -y -g
```

### Authentication

Each skill requires specific domain permissions. Log in with the necessary scopes before use:

```bash
# For daily-briefing (all 5 domains)
lark-cli auth login --domain calendar,task,mail,approval,im

# For weekly-report
lark-cli auth login --domain calendar,task,drive,vc,doc

# For action-extractor
lark-cli auth login --domain vc,drive,contact,task,im

# Or simply use --recommend for commonly used scopes
lark-cli auth login --recommend
```

## How It Works

These skills are **SKILL.md-based workflow definitions** — they teach AI Agents how to orchestrate lark-cli commands across multiple domains in a structured, step-by-step manner.

```
┌─────────────────────────────────────────────────────────┐
│                    AI Agent                              │
│  Reads SKILL.md → Understands workflow → Executes CLIs  │
└──────────┬──────────────────────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────────────────────┐
│              lark-workflow Skills                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────────────┐       │
│  │  daily   │  │  weekly  │  │    action        │       │
│  │ briefing │  │  report  │  │   extractor      │       │
│  └────┬─────┘  └────┬─────┘  └────┬─────────────┘       │
└───────┼──────────────┼─────────────┼─────────────────────┘
        │              │             │
        ▼              ▼             ▼
┌──────────────────────────────────────────────────────────┐
│              lark-cli Atomic Skills                      │
│  calendar · task · mail · approval · im · vc · docs     │
│  drive · contact · minutes · base · sheets · wiki       │
└──────────────────────────────────────────────────────────┘
```

## Contributing

We welcome contributions! See [CONTRIBUTING.md](./CONTRIBUTING.md) for guidelines.

## License

This project is licensed under the **MIT License** — see [LICENSE](./LICENSE) for details.

This software calls [Lark/Feishu Open Platform APIs](https://open.feishu.cn/) at runtime. Use of these APIs is subject to the [Lark Terms of Service](https://www.larksuite.com/user-terms-of-service) and [Privacy Policy](https://www.larksuite.com/privacy-policy).
