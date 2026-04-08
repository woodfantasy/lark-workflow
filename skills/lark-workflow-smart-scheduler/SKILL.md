---
name: lark-workflow-smart-scheduler
version: 1.0.2
description: "智能排会助手：根据参会人列表查询各人忙闲状态，自动找出共同空闲时段，支持会议室资源查询与预约， 生成最优会议时间+会议室建议并一键创建日程。当用户需要约多人会议、找共同空闲时间、预约会议室、 或问「帮我约一个 XX 的会」时使用。"
metadata:
  requires:
    bins: ["lark-cli"]
---

# 智能排会助手工作流

**CRITICAL — 开始前 MUST 先用 Read 工具读取 [`lark-shared/SKILL.md`](https://github.com/larksuite/cli/blob/main/skills/lark-shared/SKILL.md)，其中包含认证、权限处理和安全规则。**

## 适用场景

- "帮我约一个和 XX、YY 的会" / "安排一个技术评审会议"
- "找一下明天下午大家都有空的时间" / "约个 30 分钟的会"
- "看看这周哪天可以开会" / "帮我排个会"
- "张三和李四什么时候有空" / "Smart scheduling"
- "帮我创建一个会议邀请" / "约下周的复盘会"
- "帮我订个会议室" / "明天下午 17 楼有没有空的会议室"
- "帮我约个 6 人的会议室开会" / "预约会议室"

自动化「找空闲时间 → 查可用会议室 → 推荐最优时段+会议室 → 创建日程」全流程，实现"排人+排房"一站式排会。

## 前置条件

仅支持 **user 身份**。执行前确保已授权：

```bash
lark-cli auth login --domain calendar,contact
```

## 工作流

```
用户需求 ──► 解析参会人 & 时间范围 & 会议室偏好
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
                    ▼
         会议室查询 & 忙闲检查（可选）
         （查可用会议室 → 筛选空闲 → 匹配偏好）
                    │
            ┌───────┼───────┐
            ▼       ▼       ▼
          推荐1   推荐2   推荐3
     (时段+会议室) (次优)  (备选)
                    │
                    ▼
         用户选择 → calendar +create
                    （创建日程 + 邀请参会人 + 预约会议室）
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
| **会议室偏好** | "17 楼" / "能容纳 6 人" / "要有投影" | 无（不指定时跳过会议室预约） |

> **会议室触发判断**：用户提到"会议室"、"预约房间"、"订个屋子"、"线下开会"等关键词时，或指定了楼层/容量/设备偏好时，进入会议室预约流程。如果用户仅需线上会议或未提及会议室需求，跳过 Step 5。

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

**方式 A（推荐）：使用 `+freebusy` Shortcut**

```bash
# 查询多人忙闲（推荐方式，v1.0.5+）
lark-cli calendar +freebusy --start "<start_time>" --end "<end_time>" --user-ids "<user_id_1>,<user_id_2>,<user_id_3>"
```

**方式 B：使用 `+suggestion` 让 API 直接推荐空闲时段**

当用户只给了时间区间（如"明天"、"下周"）而非精确时间点时，可直接使用 `+suggestion` 获取多个推荐方案：

```bash
# 获取 AI 推荐的空闲时段（v1.0.5+）
lark-cli calendar +suggestion --start "<start_time>" --end "<end_time>" --attendee-ids "<user_id_1>,<user_id_2>" --duration-minutes 60
```

> **选择策略**：
> - 用户指定了精确时间点（"明天下午3点"） → 使用 `+freebusy` 检查冲突
> - 用户给了时间范围（"明天"、"这周"） → 使用 `+suggestion` 获取推荐，省去手动计算
> - `+suggestion` 不可用时 → 退化为 `+freebusy` + AI 手动计算

**方式 C（兜底）：使用原生 API**

```bash
# 原生 freebusy API
lark-cli calendar freebusy list --data '{
  "time_min": "<start_time>",
  "time_max": "<end_time>",
  "user_ids": ["<user_id_1>", "<user_id_2>", "<user_id_3>"]
}' --format json
```

### Step 5: 会议室查询与预约（可选 — 用户需要会议室时执行）

当用户需要预约会议室时，在计算出共同空闲时段后，查询可用会议室。

#### Step 5.1: 获取会议室列表

```bash
# 查询会议室列表
lark-cli calendar meeting_rooms list --data '{
  "page_size": 50
}' --format json
```

从返回结果中提取：
- `room_id`：会议室唯一标识
- `name`：会议室名称（如 "17F-大会议室A"）
- `capacity`：容纳人数
- `building_name` / `floor_name`：所在建筑和楼层

> **筛选规则**（按用户偏好过滤）：
> - 指定楼层 → 按 `floor_name` 过滤
> - 指定容量 → `capacity ≥ 参会人数`（默认以参会人数为最低容量要求）
> - 指定建筑 → 按 `building_name` 过滤
> - 无偏好 → 取全部可用会议室，按容量升序排列（避免大房间浪费）

#### Step 5.2: 查询会议室忙闲

对候选时段，查询候选会议室的忙闲状态：

```bash
# 查询会议室忙闲
lark-cli calendar freebusy list --data '{
  "time_min": "<slot_start_time>",
  "time_max": "<slot_end_time>",
  "room_ids": ["<room_id_1>", "<room_id_2>", "<room_id_3>"]
}' --format json
```

#### Step 5.3: 匹配最优会议室

为每个候选时段匹配一个空闲会议室，优先规则：

| 因素 | 优先级 | 说明 |
|------|--------|------|
| 容量匹配 | 最高 | 容量 ≥ 参会人数，且不过度浪费（容量 ≤ 2× 参会人数） |
| 楼层偏好 | 高 | 如用户指定楼层，完全匹配 |
| 设备需求 | 中 | 如用户提到投影/白板等，需匹配 |
| 距离最近 | 低 | 同楼层 > 相邻楼层 > 其他楼层 |

> **降级策略**：如果指定楼层/建筑无空闲会议室，自动扩大到相邻楼层或同建筑其他楼层，并在推荐中注明。

### Step 6: 计算共同空闲时段并展示推荐

**算法逻辑：**

1. 将每位参会人的忙碌时段合并为一个"已占用时间线"
2. 从工作时间中去除所有"已占用"时段
3. 筛选出 ≥ 所需会议时长的连续空闲段
4. 如需会议室：为每个候选时段匹配空闲会议室（Step 5）
5. 按以下规则排序推荐：

**排序权重：**

| 因素 | 权重 | 说明 |
|------|------|------|
| 最早可用 | 高 | 越早越好（紧急场景） |
| 工作效率时段 | 中 | 10:00-11:30、14:00-16:00 优于其他时段 |
| 避开午休 | 中 | 跨 12:00-13:00 的时段降低优先级 |
| 避开下班前 | 低 | 17:00-18:00 降低优先级 |
| 用户偏好 | 最高 | 如果用户指定"上午" / "下午"，严格遵守 |
| 会议室匹配度 | 高 | 有合适会议室的时段优先（需会议室场景） |

**展示格式（含会议室）：**

```markdown
## 🗓️ 会议排期建议

### 会议信息
- **主题**：{会议主题}
- **参会人**：{张三、李四、王五}（共 N 人）
- **时长**：{60} 分钟
- **会议室偏好**：{17 楼，≥6 人}

### ✅ 推荐时段

| # | 日期 | 时间段 | 会议室 | 容量 | 推荐理由 |
|---|------|--------|--------|------|---------|
| 1 ⭐ | 明天（周二） | 10:00-11:00 | 17F-深圳湾 | 8人 | 所有人空闲，黄金时段，楼层匹配 |
| 2 | 明天（周二） | 14:00-15:00 | 17F-前海 | 6人 | 所有人空闲，午后效率时段 |
| 3 | 后天（周三） | 09:00-10:00 | 16F-南山 ⚠️ | 10人 | 所有人空闲，但非指定楼层 |

### ⚠️ 冲突说明
- 明天上午 09:00-10:00：张三有「产品需求评审」
- 后天下午：李四有「客户拜访」（全下午不可用）
- 17F 全部会议室明天下午 15:00-17:00 已被预约

请选择一个时段（输入序号），我将创建会议邀请并预约会议室。
```

**展示格式（无会议室需求 — 纯线上模式）：**

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

**无会议室：**

```bash
# 创建日程并邀请参会人
lark-cli calendar +create \
  --summary "<会议主题>" \
  --start "<选定开始时间>" \
  --end "<选定结束时间>" \
  --attendees "<user_id_1>,<user_id_2>,<user_id_3>"
```

**含会议室预约：**

```bash
# 创建日程 + 邀请参会人 + 预约会议室
lark-cli calendar +create \
  --summary "<会议主题>" \
  --start "<选定开始时间>" \
  --end "<选定结束时间>" \
  --attendees "<user_id_1>,<user_id_2>,<user_id_3>" \
  --meeting-room "<room_id>"
```

> **注意**：如果 `+create` 不支持 `--meeting-room` 参数，使用原生 API：
> ```bash
> lark-cli calendar events create --data '{
>   "summary": "<会议主题>",
>   "start_time": {"timestamp": "<unix_ts>", "timezone": "Asia/Shanghai"},
>   "end_time": {"timestamp": "<unix_ts>", "timezone": "Asia/Shanghai"},
>   "attendees": [
>     {"type": "user", "user_id": "<user_id_1>"},
>     {"type": "user", "user_id": "<user_id_2>"},
>     {"type": "resource", "room_id": "<room_id>"}
>   ]
> }' --format json
> ```

创建成功后输出：

```markdown
## ✅ 会议已创建

- **主题**：{会议主题}
- **时间**：{YYYY-MM-DD HH:mm - HH:mm}
- **会议室**：{17F-深圳湾（8人）}  ← 仅含会议室时显示
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
| 无可用会议室 | 提示指定楼层/时段无空闲会议室，建议扩大范围或改为线上 |
| 会议室 API 不可用 | 跳过会议室预约，仅创建日程（退化为纯排人模式），在输出中标注 |
| 会议室容量不足 | 按容量升序推荐替代会议室，标注"容量不完全匹配" |

## 权限表

| 步骤 | 命令 | 所需 scope | 是否必选 |
|------|------|-----------|---------|
| 搜索用户 | `contact +search` | `contact:user.base:readonly` | ✅ 必选 |
| 查询忙闲 | `calendar freebusy list` | `calendar:calendar.freebusy:read` | ✅ 必选 |
| 查看日程 | `calendar +agenda` | `calendar:calendar.event:read` | 备选 |
| 创建日程 | `calendar +create` | `calendar:calendar.event:write` | ✅ 必选 |
| 查询会议室 | `calendar meeting_rooms list` | `calendar:meeting_room:readonly` | 可选（会议室预约时） |
| 会议室忙闲 | `calendar freebusy list` (room_ids) | `calendar:calendar.freebusy:read` | 可选（会议室预约时） |

## 参考

- [lark-shared](https://github.com/larksuite/cli/blob/main/skills/lark-shared/SKILL.md) — 认证、权限（必读）
- [lark-calendar](https://github.com/larksuite/cli/blob/main/skills/lark-calendar/SKILL.md) — `+agenda`、`+create`、freebusy、meeting_rooms 详细用法
- [lark-contact](https://github.com/larksuite/cli/blob/main/skills/lark-contact/SKILL.md) — `+search` 详细用法
