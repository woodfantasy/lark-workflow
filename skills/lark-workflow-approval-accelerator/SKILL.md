---
name: lark-workflow-approval-accelerator
version: 1.0.0
description: "审批加速器：查看我的待处理审批和已发起审批的状态，对超时未审批的实例自动发送催办消息。当用户需要催审批、查看审批进度、或问"我的审批到哪了"时使用。"
metadata:
  requires:
    bins: ["lark-cli"]
---

# 审批加速器工作流

**CRITICAL — 开始前 MUST 先用 Read 工具读取 [`lark-shared/SKILL.md`](https://github.com/larksuite/cli/blob/main/skills/lark-shared/SKILL.md)，其中包含认证、权限处理和安全规则。**

## 适用场景

- "我的审批到哪了" / "审批进度怎么样了"
- "帮我催一下审批" / "XX 审批怎么还没通过"
- "有哪些审批等着我处理" / "待我审批的有哪些"
- "催办超时的审批" / "提醒审批人处理一下"
- "审批加速" / "Approval tracking"

## 核心价值

打通 **审批查询 → 进度追踪 → 超时检测 → 催办通知** 的完整链路，避免审批卡住影响业务进度。

## 前置条件

仅支持 **user 身份**。执行前确保已授权：

```bash
lark-cli auth login --domain approval,im,contact
```

## 工作流

```
用户意图 ──► 分流
              │
    ┌─────────┼─────────┐
    ▼                    ▼
 "我发起的审批"      "待我审批的"
     │                    │
     ▼                    ▼
 approval               approval
 +my-instances           +my-tasks
 (查询我发起的)          (查询待我审批的)
     │                    │
     ▼                    ▼
 过滤 PENDING 状态    展示待处理列表
     │
     ▼
 AI 分析超时实例
 (等待时间 > 阈值)
     │
     ▼
 展示给用户确认 ──► im +messages-send（催办消息）
```

### Step 1: 理解用户意图

分两个核心功能：

| 意图 | 触发词 | 功能 |
|------|--------|------|
| **查看我发起的审批** | "我的审批到哪了"、"审批进度"、"催审批" | 追踪 + 催办 |
| **查看待我审批的** | "等我审批的"、"待处理审批" | 待办提醒 |

### Step 2A: 查看我发起的审批（追踪模式）

参考 [`lark-approval/SKILL.md`](https://github.com/larksuite/cli/blob/main/skills/lark-approval/SKILL.md)。

```bash
# 查询我发起的审批实例
lark-cli approval +my-instances --status PENDING --format json
```

从返回结果中提取每个审批的：
- `instance_id`：审批实例 ID
- `approval_name`：审批类型名称
- `title`：审批标题/摘要
- `status`：当前状态（PENDING / APPROVED / REJECTED / CANCELLED）
- `create_time`：发起时间
- `task_list`：审批节点列表（当前停在哪个节点、审批人是谁）

### Step 2B: 查看待我审批的（待办模式）

```bash
# 查询等待我审批的任务
lark-cli approval +my-tasks --status PENDING --format json
```

从返回结果中提取每个任务的：
- `instance_id`：审批实例 ID
- `approval_name`：审批类型名称
- `title`：审批标题
- `start_time`：进入该节点的时间
- `initiator`：发起人信息

### Step 3: 分析等待时间

```bash
# 获取当前时间用于计算等待时长
date "+%s"
```

**超时阈值规则：**

| 审批类型 | 超时阈值 | 说明 |
|---------|---------|------|
| 通用（默认） | 24 小时 | 超过 1 个工作日未处理 |
| 紧急审批 | 4 小时 | 标题含"紧急"、"加急" |
| 请假/报销 | 48 小时 | 非紧急行政审批 |

**等待时间计算**：`当前时间 - 审批进入当前节点的时间`

### Step 4: 展示审批状态

**追踪模式输出：**

```markdown
## 📋 我发起的审批进度（共 N 项进行中）

### ⏰ 超时审批（需催办）

| # | 审批类型 | 标题 | 当前审批人 | 等待时间 | 状态 |
|---|---------|------|-----------|---------|------|
| 1 | 报销审批 | Q1 差旅报销 ¥8,500 | 张三 | ⚠️ 3天2小时 | 催办 |
| 2 | 采购审批 | 采购办公设备 | 李四 | ⚠️ 1天6小时 | 催办 |

### ✅ 正常进行中

| # | 审批类型 | 标题 | 当前审批人 | 等待时间 | 状态 |
|---|---------|------|-----------|---------|------|
| 3 | 请假申请 | 4/10-4/11 年假 | 王五 | 5小时 | 正常 |

### 📊 近期已完成

| # | 审批类型 | 标题 | 结果 | 完成时间 |
|---|---------|------|------|---------|
| 4 | 加班申请 | 3月周末加班 | ✅ 通过 | 昨天 |
| 5 | 出差申请 | 上海出差 3天 | ✅ 通过 | 3天前 |

需要对超时审批发送催办消息吗？（输入"催办"或指定序号）
```

**待办模式输出：**

```markdown
## 📋 待我审批（共 N 项）

### ⏰ 紧急 / 等待较久

| # | 审批类型 | 发起人 | 标题 | 等待时间 | 建议 |
|---|---------|--------|------|---------|------|
| 1 | 报销审批 | 小王 | 客户拜访报销 ¥2,100 | ⚠️ 2天 | 尽快处理 |

### 📝 常规

| # | 审批类型 | 发起人 | 标题 | 等待时间 |
|---|---------|--------|------|---------|
| 2 | 请假申请 | 小李 | 4/15 事假 1 天 | 3小时 |
| 3 | 采购申请 | 小张 | 采购键盘鼠标 ¥800 | 1小时 |
```

### Step 5: 发送催办消息（用户确认后）

> **⚠️ 安全规则：必须等用户确认后才发送催办消息。**

**Step 5.1: 获取审批人信息**

```bash
# 查询审批实例详情（获取当前审批节点的审批人）
lark-cli approval instances get --params '{"instance_id":"<instance_id>"}' --format json
```

从 `task_list` 中找到状态为 `PENDING` 的节点，提取 `user_id`。

**Step 5.2: 查询审批人的联系方式**

```bash
# 获取用户信息用于发送消息
lark-cli contact +search --query "<审批人姓名>" --format json
```

**Step 5.3: 发送催办消息**

参考 [`lark-im/SKILL.md`](https://github.com/larksuite/cli/blob/main/skills/lark-im/SKILL.md)。

```bash
# 发送催办消息
lark-cli im +messages-send \
  --receive-id "<user_id>" \
  --receive-id-type "user_id" \
  --msg-type "text" \
  --content '{"text":"您好 🙏\n\n有一项审批等待您的处理：\n\n📋 类型：<审批类型>\n📝 标题：<审批标题>\n⏰ 已等待：<等待时间>\n👤 发起人：<我的名字>\n\n请您在方便时尽快处理，感谢！"}'
```

> **注意**：催办消息语气应礼貌、专业。不应连续对同一审批人重复催办（建议每 24 小时最多催办 1 次）。

### Step 6: 输出执行报告

```markdown
## 催办执行报告

| # | 审批 | 审批人 | 催办状态 |
|---|------|--------|---------|
| 1 | Q1 差旅报销 | 张三 | ✅ 已发送催办消息 |
| 2 | 采购办公设备 | 李四 | ✅ 已发送催办消息 |

💡 建议：如果 24 小时后仍未处理，可以再次催办或通过电话/当面沟通。
```

## 容错机制

| 异常场景 | 处理方式 |
|---------|---------|
| approval 域未授权 | 提示用户授权，显示授权命令 |
| 无待处理审批 | 告知"当前没有进行中的审批" |
| 审批人 user_id 无法获取 | 展示审批信息但标注"无法自动催办" |
| 消息发送失败 | 记录失败，建议用户手动沟通 |
| 审批类型无法识别 | 使用通用超时阈值（24 小时） |

## 权限表

| 步骤 | 命令 | 所需 scope | 是否必选 |
|------|------|-----------|---------|
| 我发起的审批 | `approval +my-instances` | `approval:approval:read` | ✅ 必选 |
| 待我审批 | `approval +my-tasks` | `approval:approval:read` | ✅ 必选 |
| 审批详情 | `approval instances get` | `approval:approval:read` | ✅ 必选（催办时） |
| 搜索用户 | `contact +search` | `contact:user.base:readonly` | 推荐（催办时） |
| 发送催办 | `im +messages-send` | `im:message:send_as_bot` 或 `im:message` | 推荐（催办时） |

## 参考

- [lark-shared](https://github.com/larksuite/cli/blob/main/skills/lark-shared/SKILL.md) — 认证、权限（必读）
- [lark-approval](https://github.com/larksuite/cli/blob/main/skills/lark-approval/SKILL.md) — `+my-instances`、`+my-tasks`、实例查询详细用法
- [lark-contact](https://github.com/larksuite/cli/blob/main/skills/lark-contact/SKILL.md) — `+search` 详细用法
- [lark-im](https://github.com/larksuite/cli/blob/main/skills/lark-im/SKILL.md) — 消息发送详细用法
