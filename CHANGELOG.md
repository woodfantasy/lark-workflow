# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [1.3.0] - 2026-05-25

### Changed
- **lark-cli v1.0.12~v1.0.39 compatibility** — Adapt all 15 workflows to latest upstream changes:

#### Breaking: docs API v2 migration (9 skills)
  - All `docs +fetch`, `docs +create`, `docs +update` commands now require `--api-version v2`
  - Content format flag changed from `--format markdown` to `--doc-format markdown`
  - Update mode flag changed from `--mode` to `--command` (e.g. `--command append`)
  - Affected: `daily-briefing`, `weekly-report`, `action-extractor`, `doc-summarizer`, `knowledge-qa`, `base-analytics`, `doc-template-engine`

#### Breaking: contact +search → +search-user rename (6 skills)
  - `contact +search` renamed to `contact +search-user` upstream
  - Affected: `action-extractor`, `smart-scheduler`, `approval-accelerator`, `onboarding`, `workload-balancer`

#### New shortcuts replacing raw APIs (4 skills)
  - `drive files search` → `drive +search` with flat flags (`--edited-since`, `--doc-types`, `--mine`)
  - `wiki spaces get_node` → `wiki +node-get --token`
  - `wiki spaces list` → `wiki +space-list`
  - `wiki spaces nodes list` → `wiki +node-list`
  - `calendar events patch` → `calendar +update`
  - Affected: `weekly-report`, `doc-summarizer`, `knowledge-qa`, `wiki-auditor`, `calendar-optimizer`, `onboarding`

### Updated
- README/README_zh — Atomic skills count 22 → 26 (new: `lark-apps`, `lark-markdown`, `lark-vc-agent`, `lark-okr`)

## [1.2.0] - 2026-04-15

### Changed
- **lark-cli v1.0.10~v1.0.11 compatibility** — Upgrade adapting 8 workflows to 2 upstream releases:
  - `task-prioritizer` — Integrate `+get-related-tasks` for broader task coverage and `+search` for keyword-based filtering (v1.0.11)
  - `workload-balancer` — Use `+search --assignee` to query team members' tasks, breaking the single-user limitation (v1.0.11)
  - `weekly-report` — Add `+get-related-tasks` to capture created/followed tasks for more comprehensive reports (v1.0.11)
  - `action-extractor` — Add `+search` for pre-creation dedup check and `--section-guid` for tasklist organization (v1.0.10/v1.0.11)
  - `onboarding` — Add `wiki +move` for organizing docs, wiki member operations for granting new hire access (v1.0.10)
  - `wiki-auditor` — Add `wiki +move` for fixing misplaced docs found during audit (v1.0.10)
  - `base-analytics` — Add warning for v1.0.11 null JSON object validation in base shortcuts
  - `doc-template-engine` — Add `drive +create-shortcut` for archiving batch-generated docs (v1.0.10)

### Updated
- README/README_zh — Atomic skills count 21 → 22 (new: `lark-approval`)

## [1.1.0] - 2026-04-12

### Changed
- **lark-cli v1.0.7~v1.0.9 compatibility** — Major upgrade adapting all workflows to 3 upstream releases:
  - `smart-scheduler` — Use new `calendar +room-find` shortcut (v1.0.8) replacing manual 3-step room booking; note `+create` now defaults to video meeting
  - `base-analytics` — Use `base +record-search` for keyword search (v1.0.8), record field filters, and optional `+dashboard-arrange` for Markdown dashboards
  - `doc-template-engine` — Use `base +record-search` (v1.0.8), `slides +create` for PPT generation (v1.0.9), `sheets +write-image` (v1.0.7)
  - `meeting-efficiency` — Add `minutes search` (v1.0.9) and VC calendar event relation API for enhanced note extraction
  - `action-extractor` — Add VC calendar event relation API (v1.0.7) as alternate note token source
  - `knowledge-qa` — Add advanced docs search syntax: boolean (`AND`), `intitle:` (v1.0.7)
  - `doc-summarizer` — Same advanced docs search syntax for batch mode
  - `onboarding` — Use `wiki +node-create` shortcut (v1.0.7), note `+create` video meeting default
  - `wiki-auditor` — Use `wiki +node-create` shortcut (v1.0.7) for writing audit reports to wiki

### Updated
- README/README_zh — Atomic skills count 18 → 21 (new: `lark-attendance`, `lark-slides`, `lark-whiteboard-cli`)
- Architecture diagram updated to reflect full 21-skill landscape

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
