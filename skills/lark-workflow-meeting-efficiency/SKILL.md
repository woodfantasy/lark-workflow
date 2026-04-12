---
name: lark-workflow-meeting-efficiency
version: 1.1.0
description: "会议效率分析：统计一段时间内的会议数据（总时长、参会人数、有无纪要）， 计算会议时间占比和效率指标，生成会议效率报告和优化建议。 当用户需要了解会议效率、分析会议模式、或问「这周开了多少会」时使用。"
metadata:
  requires:
    bins: ["lark-cli"]
---

# 会议效率分析工作流

**CRITICAL — 开始前 MUST 先用 Read 工具读取 [`lark-shared/SKILL.md`](https://github.com/larksuite/cli/blob/main/skills/lark-shared/SKILL.md)，其中包含认证、权限处理和安全规则。**

## 适用场景

- "这周开了多少会" / "我的会议效率怎么样"
- "分析一下最近的会议模式" / "会议时间占了多少"
- "哪个时间段会议最多" / "会议效率报告"
- "有多少会议没有纪要" / "会议质量分析"
- "Meeting efficiency analysis" / "帮我看看会议数据"

与已有 Skill 的关系：
- `meeting-summary`（官方）：**纪要汇总** — 内容维度
- `action-extractor`（已有）：**任务提取** — 执行维度
- **`meeting-efficiency`**（本 Skill）：**效率元分析** — 数据维度

三者形成完整的「会议三件套」。

## 前置条件

支持 **user 身份**。执行前确保已授权：

```bash
lark-cli auth login --domain vc,calendar
```

## 工作流

```
指定时间范围 ──► calendar +agenda（获取日程中的会议）
                       │
                       ▼
                  vc +search（获取实际参会记录）
                       │
                       ▼
                  vc +notes（检查纪要覆盖率）
                       │
                       ▼
                  AI 效率分析引擎
                  ┌────┼────┐
                  ▼    ▼    ▼
               时长  质量  节奏
               分析  分析  分析
                  │
                  ▼
            会议效率报告 + 优化建议
```

### Step 1: 确定分析时间范围

推断规则：
- "这周的会议" → 本周一 ~ 今天
- "上周" / "最近一周" → 过去 7 天
- "这个月" → 月初 ~ 今天
- "最近一个月" → 过去 30 天

```bash
# 获取时间范围
date "+%Y-%m-%dT00:00:00%z"     # 起始
date "+%Y-%m-%dT23:59:59%z"     # 结束
```

> **注意**：日期计算必须使用 `date` 命令。

### Step 2: 获取会议数据

**2.1 从日历获取已排会议：**

参考 [`lark-calendar/SKILL.md`](https://github.com/larksuite/cli/blob/main/skills/lark-calendar/SKILL.md)。

```bash
lark-cli calendar +agenda --start "<start>" --end "<end>"
```

从日程中筛选会议类事件（排除全天事件、个人日程等）：
- 有参会人的事件 → 会议
- 标题含"会议"、"review"、"sync"、"standup" → 会议
- `free_busy_status` 为 `busy` → 占用时间

**2.2 从 VC 获取实际参会记录：**

参考 [`lark-vc/SKILL.md`](https://github.com/larksuite/cli/blob/main/skills/lark-vc/SKILL.md)。

```bash
lark-cli vc +search --start "<start>" --end "<end>" --format json --page-size 50
```

VC 记录包含：
- 实际参会人数
- 会议实际时长
- 是否有录制/妙记

**2.3 检查纪要覆盖率：**

```bash
# 方式 A：通过 meeting_id 获取纪要
lark-cli vc +notes --meeting-ids "<id1>,<id2>,...,<idN>"

# 方式 B（v1.0.7+）：通过日历事件关联 API 提取纪要 token
# 当会议通过日历事件创建时，可直接从 event relation API 获取 doc_token
lark-cli vc +notes --meeting-ids "<id1>,<id2>,...,<idN>"
```

**2.4 搜索妙记/纪要（v1.0.9+，补充数据源）：**

```bash
# 搜索妙记（v1.0.9+）— 当会议未通过 vc +search 找到时可作为补充
lark-cli minutes search --data '{"query":"<关键词>"}' --format json
```

> **提示**：`minutes search` 可以按关键词搜索妙记，适合补充 vc +search 未覆盖的会议纪要。

统计有纪要的会议数量。

### Step 3: AI 效率分析引擎

**3.1 时长分析：**
- 总会议时长（小时）
- 会议时间占工作时间比例（按 8h/天计算）
- 平均每场会议时长
- 最长/最短会议
- 每日会议时长分布

**3.2 质量分析：**
- 纪要覆盖率（有纪要的会议 / 总会议数）
- 平均参会人数
- 是否有过多的大型会议（>8人）
- 「只有日程没有 VC 记录」的会议（可能取消了但没删日程）

**3.3 节奏分析：**
- 哪些天会议最密集
- 高效时段 vs 会议密集时段
- 连续会议检测（相邻会议间隔 <15 分钟）
- 午休/早晨/下班前的会议占比

### Step 4: 生成效率报告

```markdown
## 📊 会议效率报告（{开始日期} ~ {结束日期}）

### 概览指标
| 指标 | 值 | 参考基准 | 评价 |
|------|---|---------|------|
| 会议总数 | {N} 场 | — | — |
| 总耗时 | {X} 小时 | — | — |
| 占工作时间 | {P}% | <30% 健康 | {🟢/🟡/🔴} |
| 平均时长 | {M} 分钟 | 30-45min 最优 | {🟢/🟡/🔴} |
| 纪要覆盖率 | {Q}% | >80% 优秀 | {🟢/🟡/🔴} |
| 平均参会人数 | {R} 人 | 3-6 人最佳 | {🟢/🟡/🔴} |

### ⏰ 每日分布

```
周一  ████████░░  4.2h (52%)
周二  ██████░░░░  3.0h (38%)
周三  ██████████  5.5h (69%) ⚠️
周四  ████░░░░░░  2.0h (25%)
周五  ██░░░░░░░░  1.0h (13%)
```

### 🔍 模式洞察

**会议时段分布：**
| 时段 | 会议数 | 占比 | 建议 |
|------|--------|------|------|
| 09:00-10:00 | 3 | 15% | 🟡 晨间会议略多 |
| 10:00-12:00 | 8 | 40% | 🟢 黄金会议时段 |
| 12:00-13:00 | 1 | 5% | 🔴 占用午休 |
| 14:00-16:00 | 6 | 30% | 🟢 午后正常 |
| 17:00-18:00 | 2 | 10% | 🟡 临近下班 |

**连续会议检测：**
- ⚠️ 周三有 3 场连续会议（10:00-13:00），无缓冲时间
- ⚠️ 周一有 2 场会议间隔仅 5 分钟

### 📋 会议清单

| # | 日期 | 会议名称 | 时长 | 参会人数 | 有纪要 |
|---|------|---------|------|---------|--------|
| 1 | 周一 | 产品需求评审 | 60min | 8 | ✅ |
| 2 | 周一 | 技术方案讨论 | 45min | 5 | ✅ |
| 3 | 周二 | 项目进度同步 | 30min | 4 | ❌ |
| ... | ... | ... | ... | ... | ... |

### 💡 优化建议

1. **{建议 1}**：基于会议时间占比给出的建议
2. **{建议 2}**：基于连续会议检测的建议
3. **{建议 3}**：基于纪要覆盖率的建议
4. **{建议 4}**：基于参会人数分布的建议

### 📈 对比（如有历史数据）
- 对比上周：会议时长 {增加/减少} {X}%
- 纪要覆盖率 {提升/下降} {Y}%
```

## 容错机制

| 异常场景 | 处理方式 |
|---------|---------|
| vc 域未授权 | 仅使用 calendar 数据分析（无实际参会数据） |
| 时间范围内无会议 | 提示"该时间段无会议记录"，建议扩大范围 |
| vc +notes 部分失败 | 记录失败项，纪要覆盖率基于成功查询的会议计算 |
| 会议数量 >50 场 | 按重要性排列，详细表格仅展示 Top 20 |

## 权限表

| 步骤 | 命令 | 所需 scope | 是否必选 |
|------|------|-----------|---------|
| 日程数据 | `calendar +agenda` | `calendar:calendar.event:read` | ✅ 必选 |
| 会议记录 | `vc +search` | `vc:meeting:readonly` | 推荐 |
| 纪要检查 | `vc +notes` | `vc:meeting:readonly` | 推荐 |
| 妙记搜索 | `minutes search` | `minutes:minute:readonly` | 可选（补充搜索） |

## 参考

- [lark-shared](https://github.com/larksuite/cli/blob/main/skills/lark-shared/SKILL.md) — 认证、权限（必读）
- [lark-calendar](https://github.com/larksuite/cli/blob/main/skills/lark-calendar/SKILL.md) — `+agenda` 详细用法
- [lark-vc](https://github.com/larksuite/cli/blob/main/skills/lark-vc/SKILL.md) — `+search`、`+notes` 详细用法
- [lark-minutes](https://github.com/larksuite/cli/blob/main/skills/lark-minutes/SKILL.md) — `search`、妙记产物详细用法
