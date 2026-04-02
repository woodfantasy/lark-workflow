# Contributing to lark-workflow

Thank you for your interest in contributing! This guide will help you create high-quality workflow skills.

## Skill Structure

Each skill lives in its own directory under `skills/`:

```
skills/
└── lark-workflow-<name>/
    └── SKILL.md          # Required: main skill definition
```

## SKILL.md Requirements

### YAML Frontmatter (Required)

```yaml
---
name: lark-workflow-<name>
version: 1.0.0
description: "<功能描述>。当用户需要<触发场景>时使用。"
metadata:
  requires:
    bins: ["lark-cli"]
---
```

- **name** — Must start with `lark-workflow-` prefix
- **version** — Semantic versioning
- **description** — Must include trigger scenario ("当用户需要…时使用")
- **metadata.requires.bins** — Always include `"lark-cli"`

### Content Structure (Required Sections)

1. **前置条件** — Link to `lark-shared/SKILL.md` + required auth scopes
2. **适用场景** — Natural language trigger phrases
3. **工作流** — ASCII art data flow diagram + step-by-step instructions
4. **输出模板** — Structured output format
5. **权限表** — Required scopes per command
6. **参考** — Links to referenced atomic skills

### Design Principles

- **时间处理**：禁止 AI 心算日期，必须使用 `date` 命令
- **数据降级**：每个域都有无数据时的降级处理
- **安全确认**：写入操作前确认用户意图，建议 `--dry-run`
- **上下文管理**：大数据量场景提供截断策略

## Pull Request Process

1. Create your skill in `skills/lark-workflow-<name>/SKILL.md`
2. Test all CLI commands with real data
3. Submit PR with description of the workflow and use cases
4. Ensure SKILL.md passes format validation

## Code of Conduct

Be respectful, constructive, and collaborative. We're building tools to help people work better.
