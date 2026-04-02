---
name: lark-workflow-daily-briefing
version: 1.0.0
description: >
  每日全景简报：聚合日程、任务、邮件、审批、消息五大业务域信息，生成一站式每日简报。
  当用户需要了解今天的安排、查看每日概览、获取晨间简报、或问「今天有什么需要关注的」时使用。
metadata:
  requires:
    bins:
      - lark-cli
---

# 每日全景简报工作流

**CRITICAL — 开始前 MUST 先用 Read 工具读取 [`lark-shared/SKILL.md`](https://github.com/larksuite/cli/blob/main/skills/lark-shared/SKILL.md)，其中包含认证、权限处理和安全规则。**

## 适用场景

- "今天有什么安排" / "今天有哪些事情需要处理"
- "给我今天的全景简报" / "早间简报"
- "今天有什么需要关注的" / "每日概览"
- "Daily briefing" / "今天有什么紧急的事"
- "有什么未处理的审批/邮件/消息吗"

## 与 standup-report 的区别

`lark-workflow-standup-report`（官方）仅覆盖**日程 + 任务** 2 个域。本 Skill 覆盖 **5 个域**（日程 + 任务 + 邮件 + 审批 + 消息），提供真正的「全景」视图。

## 前置条件

仅支持 **user 身份**。执行前确保已授权：

```bash
lark-cli auth login --domain calendar,task,mail,approval,im
```

## 工作流

```
{日期} ─┬─► calendar +agenda        ──► 日程列表（会议/事件）
        ├─► task +get-my-tasks       ──► 未完成待办列表
        ├─► mail（搜索未读）          ──► 未读邮件列表（可选）
        ├─► approval（待处理）        ──► 待审批列表（可选）
        └─► im（搜索 @我 消息）       ──► 未读 @我 消息（可选）
                    │
                    ▼
              AI 汇总（时间转换 + 优先级排序 + 冲突检测）──► 全景简报
```

### Step 1: 确定日期

默认**今天**。推断规则："今天"→当天，"明天"→+1天，"后天"→+2天。

> **注意**：日期转换必须调用系统命令（如 `date`），不要心算。时间参数需使用 ISO 8601 格式。

```bash
# 获取当前日期（示例）
date "+%Y-%m-%dT00:00:00%z"
date "+%Y-%m-%dT23:59:59%z"
```

### Step 2: 获取日程（必选）

参考 [`lark-calendar/SKILL.md`](https://github.com/larksuite/cli/blob/main/skills/lark-calendar/SKILL.md)。

```bash
# 今天（默认，无需额外参数）
lark-cli calendar +agenda

# 指定日期
lark-cli calendar +agenda --start "<YYYY-MM-DD>T00:00:00+08:00" --end "<YYYY-MM-DD>T23:59:59+08:00"
```

输出包含：event_id、summary、start_time、end_time、free_busy_status、self_rsvp_status。

### Step 3: 获取未完成待办（必选）

参考 [`lark-task/SKILL.md`](https://github.com/larksuite/cli/blob/main/skills/lark-task/SKILL.md)。

```bash
# 获取今日到期的未完成任务
lark-cli task +get-my-tasks --due-end "<YYYY-MM-DD>T23:59:59+08:00"
```

> **注意**：不带过滤条件时可能返回大量历史待办（100KB+），摘要场景建议用 `--due-end` 过滤，或只展示近 7 天创建的任务。

### Step 4: 获取未读邮件摘要（可选）

参考 [`lark-mail/SKILL.md`](https://github.com/larksuite/cli/blob/main/skills/lark-mail/SKILL.md)。

> **降级处理**：如果用户未授权 mail 域、或不使用飞书邮箱，跳过此步，在简报中标注"邮件模块未授权"。

```bash
# 查看今日未读邮件（最近 20 封）
lark-cli mail +inbox --unread --limit 20 --format json
```

若命令报权限错误或返回空数据，直接跳过，不阻塞流程。

### Step 5: 获取待处理审批（可选）

参考 [`lark-approval/SKILL.md`](https://github.com/larksuite/cli/blob/main/skills/lark-approval/SKILL.md)。

> **降级处理**：如果用户未授权 approval 域，跳过此步，在简报中标注"审批模块未授权"。

```bash
# 查看我的待处理审批任务
lark-cli approval +my-tasks --status PENDING --format json
```

若命令报权限错误或返回空数据，直接跳过，不阻塞流程。

### Step 6: 获取未读 @我 消息（可选）

参考 [`lark-im/SKILL.md`](https://github.com/larksuite/cli/blob/main/skills/lark-im/SKILL.md)。

> **降级处理**：如果用户未授权 im 域，跳过此步。

```bash
# 搜索提到我的消息（最近 24 小时）
lark-cli im +messages-search --query "@" --start "<yesterday_timestamp>" --end "<today_timestamp>" --format json
```

> **注意**：此步仅在用户明确要求查看消息时执行。默认情况下可跳过，避免数据量过大。

### Step 7: AI 汇总生成简报

将 Step 2–6 的结果整合为以下结构：

```markdown
## {日期} 每日全景简报（{YYYY-MM-DD 星期X}）

### 📅 日程安排（共 N 场）
| 时间 | 事件 | 组织者 | 状态 |
|------|------|--------|------|
| 09:00-10:00 | 产品需求评审 | 张三 | 已接受 |
| 14:00-15:00 | 技术方案讨论 | 李四 | 待确认 |

### ✅ 待办事项（共 N 项）
- [ ] 完成 XX 设计文档（截止：今天）⚠️
- [ ] 回复客户反馈邮件（截止：明天）
- [ ] 更新项目进度表（无截止日期）

### 📧 未读邮件（共 N 封）
| 发件人 | 主题 | 时间 |
|--------|------|------|
| 王五 | Re: Q2 预算审批 | 08:30 |

*（如未授权邮箱，显示："📧 邮件模块未授权，使用 `lark-cli auth login --domain mail` 开启"）*

### ✍️ 待处理审批（共 N 项）
- 报销审批 - 张三提交 - 金额 ¥2,500（等待 2 天）
- 请假申请 - 李四提交 - 3月25日-26日（等待 1 天）

*（如未授权审批，显示："✍️ 审批模块未授权，使用 `lark-cli auth login --domain approval` 开启"）*

### 📊 小结
- 共 **N** 场会议，**M** 项待办，**P** 封未读邮件，**Q** 项待审批
- ⚠️ 日程冲突：{列出时间重叠的日程}
- ⏰ 已过期待办：{列出截止日期已过的任务}
- 📝 空闲时段：{推算可用的专注工作时段}
```

### Step 8: 生成文档（可选，用户要求时）

参考 [`lark-doc/SKILL.md`](https://github.com/larksuite/cli/blob/main/skills/lark-doc/SKILL.md)。

```bash
lark-cli docs +create --title "每日简报 (<date>)" --markdown "<内容>"
```

## 数据处理规则

1. **时间转换**：API 返回 Unix timestamp，需根据 `timezone` 字段转换为 `HH:mm` 格式
2. **RSVP 状态映射**：
   | API 值 | 显示文案 |
   |--------|---------|
   | `accept` | 已接受 |
   | `decline` | 已拒绝 |
   | `needs_action` | 待确认 |
   | `tentative` | 暂定 |
3. **日程排序**：按开始时间升序排列
4. **冲突检测**：检查相邻日程是否有时间重叠（前一个 end_time > 后一个 start_time）
5. **已拒绝日程**：标注"已拒绝"但不计入忙碌时段和冲突检测
6. **待办排序**：已过期 > 今日到期 > 明日到期 > 无截止时间
7. **域降级**：某个域执行失败或无数据时，在对应模块显示提示信息，不影响其他模块

## 权限表

| 步骤 | 命令 | 所需 scope | 是否必选 |
|------|------|-----------|---------|
| 日程 | `calendar +agenda` | `calendar:calendar.event:read` | ✅ 必选 |
| 任务 | `task +get-my-tasks` | `task:task:read` | ✅ 必选 |
| 邮件 | `mail +inbox` | `mail:mail:readonly` | 可选 |
| 审批 | `approval +my-tasks` | `approval:approval:read` | 可选 |
| 消息 | `im +messages-search` | `im:message:readonly` | 可选 |
| 文档 | `docs +create` | `docx:document:write` | 可选（生成文档时） |

## 参考

- [lark-shared](https://github.com/larksuite/cli/blob/main/skills/lark-shared/SKILL.md) — 认证、权限（必读）
- [lark-calendar](https://github.com/larksuite/cli/blob/main/skills/lark-calendar/SKILL.md) — `+agenda` 详细用法
- [lark-task](https://github.com/larksuite/cli/blob/main/skills/lark-task/SKILL.md) — `+get-my-tasks` 详细用法
- [lark-mail](https://github.com/larksuite/cli/blob/main/skills/lark-mail/SKILL.md) — 邮件操作详细用法
- [lark-approval](https://github.com/larksuite/cli/blob/main/skills/lark-approval/SKILL.md) — 审批操作详细用法
- [lark-im](https://github.com/larksuite/cli/blob/main/skills/lark-im/SKILL.md) — 消息操作详细用法
- [lark-doc](https://github.com/larksuite/cli/blob/main/skills/lark-doc/SKILL.md) — 文档创建详细用法
