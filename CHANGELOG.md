# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [1.0.3] - 2026-04-09

### Changed
- **lark-cli v1.0.6 compatibility** — Adapt to upstream features:
  - `action-extractor` — Use new `vc +recording` shortcut for `meeting_id` to `minute_token` conversion, streamlining the extraction workflow.

## [1.0.2] - 2026-04-08

### Changed
- **lark-cli v1.0.5 compatibility** — Adapt all workflows to upstream breaking changes:
  - `smart-scheduler` — Use `+freebusy` / `+suggestion` Shortcuts for intelligent scheduling; rename `+create-event` → `+create`
  - `calendar-optimizer` — Rename `+update-event` → `events patch`, `+delete-event` → `events delete`, `+reply-event` → `+rsvp`
  - `base-analytics` — Unified Base parameter `--app-token` → `--base-token`
  - `doc-template-engine` — Same `--base-token` parameter migration
  - `onboarding` — Rename `+create-event` → `+create`

## [1.0.1] - 2026-04-07

### Enhanced
- `smart-scheduler` — Added meeting room discovery and booking: query available rooms, check room freebusy, and book room alongside event creation for one-stop "people + room" scheduling

## [0.3.0] - 2026-04-03

### Added
- `lark-workflow-base-analytics` — Bitable intelligent analysis with trend detection and anomaly alerts
- `lark-workflow-onboarding` — New hire onboarding automation across 5 domains
- `lark-workflow-doc-template-engine` — Data-driven document generation from Base/Sheets templates
- `lark-workflow-meeting-efficiency` — Meeting meta-analysis with time cost and quality metrics
- `lark-workflow-workload-balancer` — Team workload heatmap and task assignment suggestions
- `lark-workflow-wiki-auditor` — Knowledge base health scoring and outdated doc detection
- `lark-workflow-calendar-optimizer` — Schedule conflict diagnosis and optimization plans

## [0.2.0] - 2026-04-03

### Added
- `lark-workflow-doc-summarizer` — Document summarizer with wiki link resolution and batch mode
- `lark-workflow-knowledge-qa` — RAG-style Q&A over Lark Wiki with source citations
- `lark-workflow-task-prioritizer` — Eisenhower matrix analysis with time-block suggestions
- `lark-workflow-smart-scheduler` — Multi-attendee free slot finder with one-click event creation
- `lark-workflow-approval-accelerator` — Approval tracking, timeout detection, and nudge messaging

## [0.1.0] - 2026-04-02

### Added
- Initial release with 3 workflow skills
- `lark-workflow-daily-briefing` — Daily panoramic briefing across 5 business domains
- `lark-workflow-weekly-report` — Automated weekly report generation
- `lark-workflow-action-extractor` — Meeting-to-task closed-loop automation
- Project documentation and contribution guidelines
