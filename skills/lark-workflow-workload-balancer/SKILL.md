---
name: lark-workflow-workload-balancer
version: 1.1.0
description: "团队负载均衡器：获取团队成员的任务量、日程密度和审批量，综合评估每个人的工作负载， 生成团队负载热力图和任务分配建议。 当用户需要了解团队工作量分布、或问「谁比较空可以接新任务」时使用。"
metadata:
  requires:
    bins: ["lark-cli"]
---

# 团队负载均衡器工作流

**CRITICAL — 开始前 MUST 先用 Read 工具读取 [`lark-shared/SKILL.md`](https://github.com/larksuite/cli/blob/main/skills/lark-shared/SKILL.md)，其中包含认证、权限处理和安全规则。**

## 适用场景

- "看看团队每个人的工作量" / "谁比较空可以接新任务"
- "团队负载均衡" / "工作量分析"
- "谁现在最忙" / "任务分配建议"
- "团队 Workload 报告" / "人力分配情况"
- "这个任务该分给谁" / "Team workload balancer"

与已有 `task-prioritizer`（个人视角：我的任务怎么排）互补，本 Skill 面向**团队管理者视角**：看清每个人的负载，合理分配新任务。

## 前置条件

支持 **user 身份**。执行前确保已授权：

```bash
lark-cli auth login --domain task,calendar,contact
```

> **注意**：查看他人任务和日程需要相应权限。如果无法获取某位成员的数据，将标注"数据不可用"。

## 工作流

```
团队成员列表 ──► contact +search（逐个确认 user_id）
                       │
          ┌────────────┼────────────┐
          ▼            ▼            ▼
    task（各成员    calendar       approval
     待办数量）    （日程密度）   （审批量，可选）
          │            │            │
          └────────────┼────────────┘
                       ▼
                AI 综合评估
                       │
            ┌──────────┼──────────┐
            ▼          ▼          ▼
        负载热力图  任务分配建议  风险预警
```

### Step 1: 确定团队成员列表

**方式 A — 用户指定成员名单：**

用户说"看看张三、李四、王五的工作量"，逐个搜索确认：

```bash
lark-cli contact +search --query "<姓名>" --format json
```

**方式 B — 按部门获取：**

```bash
# 获取部门成员列表
lark-cli contact users find_by_department --params '{"department_id":"<dept_id>"}' --format json
```

> **注意**：获取部门成员需要管理权限。如果无权限，提示用户手动列出团队成员。

### Step 2: 获取各成员任务量

参考 [`lark-task/SKILL.md`](https://github.com/larksuite/cli/blob/main/skills/lark-task/SKILL.md)。

**方式 A — 按负责人搜索任务（推荐，v1.0.11+）：**

```bash
# 按负责人搜索未完成任务（逐人查询）
lark-cli task +search --assignee "<user_open_id>" --completed=false --format json

# 搜索本周到期的任务
lark-cli task +search --assignee "<user_open_id>" --completed=false --due "-0d,+7d" --format json
```

> **提示**：v1.0.11 新增的 `+search` 支持 `--assignee` 参数按负责人查询，突破了 `+get-my-tasks` 只能查自己任务的限制。需要每位成员的 `open_id`（通过 Step 1 的 `contact +search` 获取）。

**方式 B — 仅查当前用户任务（降级方案）：**

```bash
# 获取当前登录用户的未完成任务
lark-cli task +get-my-tasks --format json
```

> **⚠️ 权限限制**：如果 `+search --assignee` 无权限查询他人任务，退化为用户手动输入各成员任务数据，或将结果标注为"数据受限"。

从任务数据中提取：
- 未完成任务数
- 本周到期任务数
- 已过期任务数
- 任务总预估工时（如果有工时字段）

### Step 3: 获取各成员日程密度

参考 [`lark-calendar/SKILL.md`](https://github.com/larksuite/cli/blob/main/skills/lark-calendar/SKILL.md)。

```bash
# 查询忙闲状态（freebusy，推荐方式）
lark-cli calendar freebusy list --data '{
  "time_min": "<本周一>",
  "time_max": "<本周日>",
  "user_ids": ["<user_id_1>", "<user_id_2>", "<user_id_3>"]
}' --format json
```

或逐人查询日程：
```bash
lark-cli calendar +agenda --start "<start>" --end "<end>"
```

从忙闲数据中计算：
- 本周会议总时长
- 会议占工作时间比例
- 空闲时段总量

### Step 4: AI 综合评估

**负载评分模型：**

```
负载分 = 任务权重 × 任务分 + 会议权重 × 会议分

任务分 = (未完成任务数 × 1 + 本周到期 × 2 + 已过期 × 3) / 基准值
会议分 = 会议时间占工作时间比例 / 30%

基准值 = 团队平均任务数 × 1.2
```

**负载等级：**

| 等级 | 分数范围 | 显示 | 说明 |
|------|---------|------|------|
| 低 | 0 - 0.5 | 🟢 | 有较多空余，可接新任务 |
| 适中 | 0.5 - 0.8 | 🟡 | 正常负载 |
| 高 | 0.8 - 1.2 | 🟠 | 接近满载 |
| 过载 | > 1.2 | 🔴 | 超负荷，需要减压 |

### Step 5: 生成报告

```markdown
## 👥 团队负载报告（{日期}）

### 负载热力图

| 成员 | 待办 | 本周到期 | 已过期 | 会议占比 | 负载 |
|------|------|---------|--------|---------|------|
| 张三 | 12 | 5 | 2 | 45% | 🔴 过载 |
| 李四 | 8 | 3 | 0 | 30% | 🟡 适中 |
| 王五 | 4 | 1 | 0 | 20% | 🟢 低 |
| 小赵 | 9 | 4 | 1 | 55% | 🔴 过载 |
| 小钱 | 5 | 2 | 0 | 15% | 🟢 低 |

### 📊 负荷分布
```
张三  ████████████████░░░░  80%  🔴
小赵  ██████████████████░░  90%  🔴
李四  ████████████░░░░░░░░  60%  🟡
小钱  ████████░░░░░░░░░░░░  40%  🟢
王五  ██████░░░░░░░░░░░░░░  30%  🟢
```

### ⚠️ 风险预警
- 🔴 **张三** 有 2 项已过期任务，且本周会议占比 45%，建议减轻负担
- 🔴 **小赵** 会议占比 55%，几乎无法专注处理任务
- 🟡 团队平均负载 {X}%，整体偏{高/正常/低}

### 💡 任务分配建议

如果有新任务需要分配：
1. **优先考虑**：🟢 王五（负载最低，空闲时间充足）
2. **次选**：🟢 小钱（负载较低，本周日程稀疏）
3. **避免分配给**：🔴 张三、小赵（已过载）

### 📋 建议行动
1. 与张三沟通，协助处理 2 项过期任务或延期
2. 尝试减少小赵的会议（建议取消非必要参加的会议）
3. 将张三的 {任务X} 转移给王五
```

## 容错机制

| 异常场景 | 处理方式 |
|---------|---------|
| 无法查询他人任务 | 请用户手动输入各成员任务数，或仅基于日程分析 |
| freebusy 无权限 | 退化为仅任务维度分析 |
| 部门查询无权限 | 请用户手动列出团队成员名单 |
| 某成员数据不可用 | 标注"数据不可用"，其余成员正常分析 |

## 权限表

| 步骤 | 命令 | 所需 scope | 是否必选 |
|------|------|-----------|---------|
| 搜索用户 | `contact +search` | `contact:user.base:readonly` | ✅ 必选 |
| 部门成员 | `contact users find_by_department` | `contact:user.employee:readonly` | 可选 |
| 搜索任务 | `task +search` | `task:task:read` | 推荐（v1.0.11+） |
| 我的任务 | `task +get-my-tasks` | `task:task:read` | 备选 |
| 忙闲查询 | `calendar freebusy list` | `calendar:calendar.freebusy:read` | 推荐 |
| 日程查询 | `calendar +agenda` | `calendar:calendar.event:read` | 备选 |

## 参考

- [lark-shared](https://github.com/larksuite/cli/blob/main/skills/lark-shared/SKILL.md) — 认证、权限（必读）
- [lark-task](https://github.com/larksuite/cli/blob/main/skills/lark-task/SKILL.md) — `+get-my-tasks` 详细用法
- [lark-calendar](https://github.com/larksuite/cli/blob/main/skills/lark-calendar/SKILL.md) — freebusy、`+agenda` 详细用法
- [lark-contact](https://github.com/larksuite/cli/blob/main/skills/lark-contact/SKILL.md) — `+search` 详细用法
