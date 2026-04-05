---
name: lark-workflow-weekly-report
version: 1.0.0
description: "周报自动生成：聚合一周的日程、已完成任务、会议纪要、文档活动，自动生成结构化周报。 当用户需要写周报、生成工作总结、回顾本周工作、或问「帮我写周报」时使用。"
metadata:
  requires:
    bins: ["lark-cli"]
---

# 周报自动生成工作流

**CRITICAL — 开始前 MUST 先用 Read 工具读取 [`lark-shared/SKILL.md`](https://github.com/larksuite/cli/blob/main/skills/lark-shared/SKILL.md)，其中包含认证、权限处理和安全规则。**

## 适用场景

- "帮我写周报" / "生成本周工作总结"
- "这周做了什么" / "整理本周工作"
- "周报" / "weekly report"
- "回顾上周工作" / "上周总结"
- "帮我总结这周完成了什么"

- `standup-report`（官方）：聚合**当日**日程 + 待办 → 时间导向
- `meeting-summary`（官方）：仅汇总会议纪要
- **本 Skill**：聚合**整周**多域数据 → **结果导向**（你完成了什么，而不只是你参加了什么）

## 前置条件

仅支持 **user 身份**。执行前确保已授权：

```bash
lark-cli auth login --domain calendar,task,drive,vc,doc
```

## 工作流

```
{周一~今天} ─┬─► calendar +agenda         ──► 本周会议列表
             ├─► task +get-my-tasks        ──► 本周已完成 & 进行中任务
             ├─► drive（搜索文档活动）      ──► 本周编辑/创建的文档（可选）
             └─► vc +search → vc +notes    ──► 本周会议纪要（可选）
                         │
                         ▼
                   AI 聚合分析
                         │
            ┌────────────┼────────────┐
            ▼            ▼            ▼
        本周完成     下周计划      风险与问题
            │
            ▼
        周报文档（可选：docs +create 写回飞书）
```

### Step 1: 确定周报时间范围

推断规则：
- "周报" / "本周" → 本周一 00:00:00 ~ 当前时间
- "上周周报" / "上周" → 上周一 00:00:00 ~ 上周日 23:59:59

> **注意**：日期转换必须调用系统命令，不要心算。

```bash
# 获取本周一的日期（macOS）
date -v-monday "+%Y-%m-%dT00:00:00%z"
# 获取当前时间
date "+%Y-%m-%dT%H:%M:%S%z"

# 获取上周一和上周日（macOS）
date -v-monday -v-7d "+%Y-%m-%dT00:00:00%z"
date -v-sunday -v-7d "+%Y-%m-%dT23:59:59%z"
```

### Step 2: 获取本周日程

参考 [`lark-calendar/SKILL.md`](https://github.com/larksuite/cli/blob/main/skills/lark-calendar/SKILL.md)。

```bash
lark-cli calendar +agenda --start "<monday>T00:00:00+08:00" --end "<today>T23:59:59+08:00"
```

从日程中提取：
- 会议总数和总时长
- 关键会议列表（去除日常站会等重复性会议）
- 参与的重要决策

### Step 3: 获取任务完成情况

参考 [`lark-task/SKILL.md`](https://github.com/larksuite/cli/blob/main/skills/lark-task/SKILL.md)。

```bash
# 获取所有我的任务（包含已完成和未完成）
lark-cli task +get-my-tasks --page-all --format json
```

从任务中提取：
- **已完成任务**：本周内从"未完成"变为"已完成"的任务（根据完成时间判断）
- **进行中任务**：仍未完成但本周有更新的任务
- **逾期任务**：截止日期在本周内但未完成的任务

> **注意**：`--page-all` 可能返回大量数据。如果任务量过大（>50 条），建议提示用户当前返回的是全量数据，AI 将仅摘取本周相关任务。

### Step 4: 获取文档活动（可选）

参考 [`lark-drive/SKILL.md`](https://github.com/larksuite/cli/blob/main/skills/lark-drive/SKILL.md)。

> **降级处理**：如果 drive 域未授权或返回为空，跳过此步。

```bash
# 搜索最近编辑过的文档
lark-cli drive files search --data '{"search_key":"","count":20,"order_by":"EditedTime","asc":false}' --format json
```

从结果中过滤出本周编辑/创建的文档（根据 `edit_time` / `create_time` 判断）。

### Step 5: 获取会议纪要摘要（可选）

参考 [`lark-vc/SKILL.md`](https://github.com/larksuite/cli/blob/main/skills/lark-vc/SKILL.md)。

> **降级处理**：如果 vc 域未授权或返回为空，跳过此步。

```bash
# 搜索本周会议
lark-cli vc +search --start "<monday>" --end "<today>" --format json --page-size 30

# 获取会议纪要文档 token
lark-cli vc +notes --meeting-ids "id1,id2,...,idN"
```

> **注意**：不需要读取纪要正文全文，仅需纪要标题和文档链接。AI 根据会议标题和纪要标题即可推断会议内容概要。

### Step 6: AI 生成周报

将 Step 2–5 的结果整合为以下结构：

```markdown
## 周报（{YYYY-MM-DD} ~ {YYYY-MM-DD}）

### 📌 本周概要
> 一段话总结本周核心工作成果（由 AI 根据以下数据生成）

### ✅ 本周完成
1. **{任务/事项 1}**
   - 完成时间：YYYY-MM-DD
   - 相关文档：[文档名](链接)（如有）
   
2. **{任务/事项 2}**
   - 完成时间：YYYY-MM-DD

### 🔄 进行中
1. **{任务/事项}** — 当前进度描述 — 预计完成日期

### 📅 主要会议
| 日期 | 会议 | 关键结论 | 纪要链接 |
|------|------|---------|---------|
| 周一 | 产品需求评审 | 确定 Q2 方向 | [链接] |
| 周三 | 技术方案讨论 | 采用方案 B | [链接] |

### 📄 文档产出
- [X 设计文档](链接) — 新建
- [Y 需求文档](链接) — 更新

### 📋 下周计划
1. {基于进行中任务和会议决策推导的计划 1}
2. {计划 2}
3. {计划 3}

### ⚠️ 风险与问题
- {基于逾期任务发现的风险}
- {需要协调解决的问题}
```

**生成规则：**

1. **本周完成**：优先从已完成任务列表提取，补充从文档产出和会议结论中推断的成果
2. **下周计划**：从「进行中任务」的待完成部分 + 会议纪要中提取的后续行动项推导
3. **风险与问题**：从逾期任务、未完成的关键任务中提取
4. **概要**：根据全部数据生成一段不超过 100 字的总结
5. **会议压缩**：如果会议超过 10 场，仅列出最重要的 5-8 场，其余折叠为"其他 N 场常规会议"

### Step 7: 生成文档（可选，用户要求时）

参考 [`lark-doc/SKILL.md`](https://github.com/larksuite/cli/blob/main/skills/lark-doc/SKILL.md)。

```bash
lark-cli docs +create --title "周报 (<start> ~ <end>)" --markdown "<内容>"
# 或追加到已有文档
lark-cli docs +update --doc "<url_or_token>" --mode append --markdown "<内容>"
```

## 权限表

| 步骤 | 命令 | 所需 scope | 是否必选 |
|------|------|-----------|---------|
| 日程 | `calendar +agenda` | `calendar:calendar.event:read` | ✅ 必选 |
| 任务 | `task +get-my-tasks` | `task:task:read` | ✅ 必选 |
| 文档活动 | `drive files search` | `drive:drive:readonly` | 可选 |
| 会议记录 | `vc +search` / `vc +notes` | `vc:meeting:readonly` | 可选 |
| 生成文档 | `docs +create` / `docs +update` | `docx:document:write` | 可选（写回时） |

## 参考

- [lark-shared](https://github.com/larksuite/cli/blob/main/skills/lark-shared/SKILL.md) — 认证、权限（必读）
- [lark-calendar](https://github.com/larksuite/cli/blob/main/skills/lark-calendar/SKILL.md) — `+agenda` 详细用法
- [lark-task](https://github.com/larksuite/cli/blob/main/skills/lark-task/SKILL.md) — `+get-my-tasks` 详细用法
- [lark-drive](https://github.com/larksuite/cli/blob/main/skills/lark-drive/SKILL.md) — 文件搜索详细用法
- [lark-vc](https://github.com/larksuite/cli/blob/main/skills/lark-vc/SKILL.md) — `+search`、`+notes` 详细用法
- [lark-doc](https://github.com/larksuite/cli/blob/main/skills/lark-doc/SKILL.md) — `+create`、`+update` 详细用法
