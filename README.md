[中文版](./README_zh.md) | [English](./README.md)

# lark-workflow

> 15 cross-domain workflow skills for [Lark/Feishu CLI](https://github.com/larksuite/cli) — orchestrate multiple business domains into powerful automated workflows.

While [lark-cli](https://github.com/larksuite/cli) provides 26 atomic skills for individual domains (Calendar, IM, Docs, Tasks, etc.), **lark-workflow** combines them into multi-step, cross-domain workflows that solve real-world productivity scenarios.

## Quick Start

### 1. Install

**Option A: via npx (GitHub source)**

```bash
# Install lark-cli (if not already installed)
npm install -g @larksuite/cli

# Install official lark-cli skills (required dependency)
npx skills add larksuite/cli -y -g

# Install lark-workflow skills
npx skills add woodfantasy/lark-workflow -y -g
```

**Option B: via OpenClaw (ClawHub)**

```bash
openclaw install lark-workflow
```

### 2. Authenticate

```bash
# Recommended: authorize all commonly used domains at once
lark-cli auth login --recommend
```

### 3. Use

Simply tell your AI Agent what you need in natural language:

```
💬 "Give me today's briefing"           → daily-briefing
💬 "Write my weekly report"             → weekly-report
💬 "Extract action items from meeting"  → action-extractor
💬 "Summarize this document"            → doc-summarizer
💬 "What does the wiki say about X?"    → knowledge-qa
💬 "Prioritize my tasks for today"      → task-prioritizer
💬 "Schedule a meeting with X and Y"    → smart-scheduler
💬 "Where's my approval at?"            → approval-accelerator
💬 "Analyze this Bitable"               → base-analytics
💬 "New hire X is joining"              → onboarding
💬 "Generate docs from template"        → doc-template-engine
💬 "How efficient are my meetings?"     → meeting-efficiency
💬 "Who has bandwidth on the team?"     → workload-balancer
💬 "Audit the wiki for stale docs"      → wiki-auditor
💬 "Optimize my calendar"               → calendar-optimizer
```

The AI Agent reads the corresponding SKILL.md, understands the workflow, and executes the right lark-cli commands automatically.

## Skills

### 🔄 Personal Productivity

| Skill | Description | Domains |
|-------|-------------|---------|
| [`daily-briefing`](./skills/lark-workflow-daily-briefing/SKILL.md) | Panoramic daily briefing across 5 domains | Calendar, Task, Mail, Approval, IM |
| [`weekly-report`](./skills/lark-workflow-weekly-report/SKILL.md) | Automated weekly report from tasks, meetings, and docs | Calendar, Task, Drive, VC, Docs |
| [`task-prioritizer`](./skills/lark-workflow-task-prioritizer/SKILL.md) | Eisenhower matrix + time-block suggestions | Task, Calendar |
| [`calendar-optimizer`](./skills/lark-workflow-calendar-optimizer/SKILL.md) | Conflict diagnosis and schedule optimization | Calendar |

### 🤝 Team Collaboration

| Skill | Description | Domains |
|-------|-------------|---------|
| [`smart-scheduler`](./skills/lark-workflow-smart-scheduler/SKILL.md) | Multi-attendee free slot finder + room booking + event creation | Calendar, Contact |
| [`action-extractor`](./skills/lark-workflow-action-extractor/SKILL.md) | Meeting → notes → tasks → notifications closed loop | VC, Minutes, Note, Docs, Contact, Task, IM |
| [`onboarding`](./skills/lark-workflow-onboarding/SKILL.md) | New hire automation across 5 domains | Contact, IM, Task, Wiki, Calendar |
| [`workload-balancer`](./skills/lark-workflow-workload-balancer/SKILL.md) | Team workload heatmap + assignment suggestions | Task, Calendar, Contact |
| [`approval-accelerator`](./skills/lark-workflow-approval-accelerator/SKILL.md) | Approval tracking + timeout nudge messaging | Approval, IM, Contact |

### 📊 Data & Analytics

| Skill | Description | Domains |
|-------|-------------|---------|
| [`base-analytics`](./skills/lark-workflow-base-analytics/SKILL.md) | Bitable trend analysis, anomaly detection, insights | Base |
| [`meeting-efficiency`](./skills/lark-workflow-meeting-efficiency/SKILL.md) | Meeting time cost, note coverage, pattern analysis | VC, Calendar, Minutes, Note |
| [`doc-template-engine`](./skills/lark-workflow-doc-template-engine/SKILL.md) | Data-driven batch document generation | Docs, Base, Sheets |

### 📚 Knowledge Management

| Skill | Description | Domains |
|-------|-------------|---------|
| [`doc-summarizer`](./skills/lark-workflow-doc-summarizer/SKILL.md) | Structured document/wiki summaries | Docs, Wiki, Drive |
| [`knowledge-qa`](./skills/lark-workflow-knowledge-qa/SKILL.md) | RAG-style Q&A with source citations | Wiki, Docs, Drive |
| [`wiki-auditor`](./skills/lark-workflow-wiki-auditor/SKILL.md) | Knowledge base health scoring + stale doc detection | Wiki, Drive |

## How It Works

These skills are **SKILL.md-based workflow definitions**. Each SKILL.md file teaches an AI Agent how to orchestrate multiple lark-cli commands in a structured, step-by-step manner.

```
┌─────────────────────────────────────────────────────────────────────┐
│                          AI Agent                                    │
│     You say: "Give me today's briefing"                             │
│     Agent reads: daily-briefing/SKILL.md                            │
│     Agent executes: calendar → task → mail → approval → AI summary  │
└──────────┬──────────────────────────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    lark-workflow Skills (15)                          │
│                                                                      │
│  Personal          Team             Analytics       Knowledge        │
│  ┌────────────┐  ┌──────────────┐  ┌────────────┐  ┌────────────┐  │
│  │briefing    │  │scheduler     │  │base-analyt.│  │doc-summary │  │
│  │weekly-rpt  │  │action-extr.  │  │meeting-eff.│  │knowledge-qa│  │
│  │task-prior. │  │onboarding    │  │doc-template│  │wiki-auditor│  │
│  │cal-optimiz.│  │workload-bal. │  └────────────┘  └────────────┘  │
│  └────────────┘  │approval-acc. │                                    │
│                  └──────────────┘                                    │
└──────────┬──────────────────────────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────────────────────────────┐
│                  lark-cli Atomic Skills (27)                          │
│  calendar · task · mail · approval · im · vc · docs · drive         │
│  contact · minutes · note · base · sheets · wiki · event · whiteboard │
│  slides · attendance · openapi-explorer · skill-maker · shared      │
│  apps · markdown · okr · vc-agent                                   │
└─────────────────────────────────────────────────────────────────────┘
```

### What makes a SKILL.md?

Each skill follows the [lark-skill-maker](https://github.com/larksuite/cli/tree/main/skills/lark-skill-maker) standard:

- **YAML frontmatter** — name, version, description, dependencies
- **Trigger phrases** — natural language patterns that activate the skill
- **Workflow steps** — Step 1 → 2 → 3... with exact CLI commands
- **Output template** — structured Markdown format for consistent results
- **Permission table** — required OAuth scopes per command
- **Error handling** — graceful degradation when a domain is unavailable

## Authentication Reference

Each skill lists required scopes in its SKILL.md. Here's a quick reference:

| Skill | Auth Command |
|-------|-------------|
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

> **Tip:** Use `lark-cli auth login --recommend` to authorize the most commonly used scopes in one step.

> **Least-privilege alternative:** If you only use a subset of skills, authorize only the domains you need — each row above shows the minimum scopes for that skill.

## Security & Privacy

- **No secrets stored** — lark-workflow is instruction-only (SKILL.md files). It never reads, stores, or transmits credentials. Authentication is handled entirely by `lark-cli auth`, which stores tokens in the system keychain.
- **No code execution** — Installation adds only Markdown skill files. No scripts, binaries, or background processes are installed.
- **User confirmation required** — All write operations (create events, send messages, create tasks, etc.) require explicit user confirmation before execution.
- **Scope transparency** — Each skill declares its required OAuth scopes in a permission table. Use per-skill `--domain` auth to grant only the minimum permissions needed.
- **Graceful degradation** — Optional domains that fail or are unauthorized are skipped with a notice, never blocking the main workflow.

## Contributing

We welcome contributions! See [CONTRIBUTING.md](./CONTRIBUTING.md) for guidelines.

## License

This project is licensed under the **MIT License** — see [LICENSE](./LICENSE) for details.

This software calls [Lark/Feishu Open Platform APIs](https://open.feishu.cn/) at runtime. Use of these APIs is subject to the [Lark Terms of Service](https://www.larksuite.com/user-terms-of-service) and [Privacy Policy](https://www.larksuite.com/privacy-policy).
