---
name: lark-workflow-smart-scheduler
version: 1.0.0
description: >
  智能排会助手：根据参会人列表查询各人忙闲状态，自动找出共同空闲时段，
  生成最优会议时间建议并一键创建日程。当用户需要约多人会议、找共同空闲时间、
  或问「帮我约一个 XX 的会」时使用。
metadata:
  requires:
    bins:
      - lark-cli
---

# 智能排会助手工作流

**CRITICAL — 开始前 MUST 先用 Read 工具读取 [`lark-shared/SKILL.md`](https://github.com/larksuite/cli/blob/main/skills/lark-shared/SKILL.md)，其中包含认证、权限处理和安全规则。**

## 适用场景

- "帮我约一个和 XX、YY 的会" / "安排一个技术评审会议"
- "找一下明天下午大家都有空的时间" / "约个 30 分钟的会"
- "看看这周哪天可以开会" / "帮我排个会"
- "张三和李四什么时候有空" / "Smart scheduling"
- "帮我创建一个会议邀请" / "约下周的复盘会"

## 核心价值

自动化「找空闲时间 → 推荐最优时段 → 创建日程」全流程，避免手动逐个查看日历。

## 前置条件

仅支持 **user 身份**。执行前确保已授权：

```bash
lark-cli auth login --domain calendar,contact
```

## 工作流

```
用户需求 ──► 解析参会人 & 时间范围
                    │
                    ├─► contact +search（解析姓名 → user_id）
                    │
                    ▼
            calendar freebusy list
          （查询每人忙闲状态）
                    │
                    ▼
         AI 计算共同空闲时段
                    │
            ┌───────┼───────┐
            ▼       ▼       ▼
          推荐1   推荐2   推荐3
         (最优)  (次优)  (备选)
                    │
                    ▼
         用户选择 → calendar +create-event
                    （创建日程 + 邀请参会人）
```

### Step 1: 解析用户需求

从自然语言中提取：

| 信息项 | 示例 | 默认值 |
|--------|------|--------|
| **参会人** | "和张三、李四" | 无（必须指定） |
| **时长** | "30 分钟" / "1 小时" | 60 分钟 |
| **时间范围** | "明天下午" / "这周" / "下周一到周三" | 未来 3 个工作日 |
| **会议主题** | "技术评审" / "需求对齐" | 无 |
| **偏好** | "尽量上午" / "避开午饭时间" | 无 |

### Step 2: 解析参会人身份

参考 [`lark-contact/SKILL.md`](https://github.com/larksuite/cli/blob/main/skills/lark-contact/SKILL.md)。

```bash
# 按姓名搜索用户
lark-cli contact +search --query "张三" --format json
```

从返回结果中提取：
- `user_id`：用于查询忙闲和创建日程
- `name`：确认是否为目标用户

> **多个匹配时**：列出供用户选择（"找到 2 位'张三'，请确认是哪位"）。

### Step 3: 确定搜索时间范围

```bash
# 获取搜索范围（示例：未来 3 个工作日）
date "+%Y-%m-%dT09:00:00%z"    # 起始
date -v+3d "+%Y-%m-%dT18:00:00%z"  # 结束
```

> **注意**：日期计算必须使用 `date` 命令，不可心算。

**工作时间假设**：
- 默认工作日 09:00-18:00
- 午休 12:00-13:00 标记为不可用
- 周末不纳入搜索范围（除非用户明确要求）

### Step 4: 查询忙闲状态

参考 [`lark-calendar/SKILL.md`](https://github.com/larksuite/cli/blob/main/skills/lark-calendar/SKILL.md)。

```bash
# 查询多人忙闲（freebusy）
lark-cli calendar freebusy list --data '{
  "time_min": "<start_time>",
  "time_max": "<end_time>",
  "user_ids": ["<user_id_1>", "<user_id_2>", "<user_id_3>"]
}' --format json
```

返回结果包含每人在指定时间范围内的忙碌时段。

> **备选方案**：如果 freebusy API 不可用，退化为逐个查询每人的 `+agenda`：
> ```bash
> lark-cli calendar +agenda --user "<user_id>" --start "<start>" --end "<end>"
> ```

### Step 5: 计算共同空闲时段

**算法逻辑：**

1. 将每位参会人的忙碌时段合并为一个"已占用时间线"
2. 从工作时间中去除所有"已占用"时段
3. 筛选出 ≥ 所需会议时长的连续空闲段
4. 按以下规则排序推荐：

**排序权重：**

| 因素 | 权重 | 说明 |
|------|------|------|
| 最早可用 | 高 | 越早越好（紧急场景） |
| 工作效率时段 | 中 | 10:00-11:30、14:00-16:00 优于其他时段 |
| 避开午休 | 中 | 跨 12:00-13:00 的时段降低优先级 |
| 避开下班前 | 低 | 17:00-18:00 降低优先级 |
| 用户偏好 | 最高 | 如果用户指定"上午" / "下午"，严格遵守 |

### Step 6: 展示推荐时段

```markdown
## 🗓️ 会议排期建议

### 会议信息
- **主题**：{会议主题}
- **参会人**：{张三、李四、王五}（共 N 人）
- **时长**：{60} 分钟

### ✅ 推荐时段

| # | 日期 | 时间段 | 推荐理由 |
|---|------|--------|---------|
| 1 ⭐ | 明天（周二） | 10:00-11:00 | 所有人空闲，黄金工作时段 |
| 2 | 明天（周二） | 14:00-15:00 | 所有人空闲，午后效率时段 |
| 3 | 后天（周三） | 09:00-10:00 | 所有人空闲，但较早 |

### ⚠️ 冲突说明
- 明天上午 09:00-10:00：张三有「产品需求评审」
- 后天下午：李四有「客户拜访」（全下午不可用）

请选择一个时段（输入序号），我将创建会议邀请。
```

### Step 7: 创建日程（用户确认后）

参考 [`lark-calendar/SKILL.md`](https://github.com/larksuite/cli/blob/main/skills/lark-calendar/SKILL.md)。

> **⚠️ 安全规则：必须等用户确认选择后才创建日程。**

```bash
# 创建日程并邀请参会人
lark-cli calendar +create-event \
  --summary "<会议主题>" \
  --start "<选定开始时间>" \
  --end "<选定结束时间>" \
  --attendees "<user_id_1>,<user_id_2>,<user_id_3>"
```

创建成功后输出：

```markdown
## ✅ 会议已创建

- **主题**：{会议主题}
- **时间**：{YYYY-MM-DD HH:mm - HH:mm}
- **参会人**：{张三、李四、王五}
- **邀请状态**：已发送日程邀请

参会人将收到飞书日历通知。
```

## 容错机制

| 异常场景 | 处理方式 |
|---------|---------|
| 人名搜索无结果 | 提示用户确认姓名或提供 user_id |
| 多个同名用户 | 列出供用户选择 |
| 无共同空闲时段 | 展示最接近的选项（仅 1-2 人有冲突的时段），建议用户协调 |
| freebusy API 不可用 | 退化为 +agenda 逐人查询 |
| 日程创建失败 | 显示错误信息，建议用户手动创建 |
| 时间范围内全无空闲 | 建议扩大搜索范围（多查几天），或减少参会人数 |

## 权限表

| 步骤 | 命令 | 所需 scope | 是否必选 |
|------|------|-----------|---------|
| 搜索用户 | `contact +search` | `contact:user.base:readonly` | ✅ 必选 |
| 查询忙闲 | `calendar freebusy list` | `calendar:calendar.freebusy:read` | ✅ 必选 |
| 查看日程 | `calendar +agenda` | `calendar:calendar.event:read` | 备选 |
| 创建日程 | `calendar +create-event` | `calendar:calendar.event:write` | ✅ 必选 |

## 参考

- [lark-shared](https://github.com/larksuite/cli/blob/main/skills/lark-shared/SKILL.md) — 认证、权限（必读）
- [lark-calendar](https://github.com/larksuite/cli/blob/main/skills/lark-calendar/SKILL.md) — `+agenda`、`+create-event`、freebusy 详细用法
- [lark-contact](https://github.com/larksuite/cli/blob/main/skills/lark-contact/SKILL.md) — `+search` 详细用法
