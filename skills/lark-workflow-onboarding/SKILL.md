---
name: lark-workflow-onboarding
version: 1.2.0
description: "新人入职自动化：根据新员工信息，自动执行拉入群聊、创建入职任务清单、发送知识库必读文档、 安排入职培训日程等操作，一个 Skill 编排 5 个业务域。 当用户需要为新同事准备入职流程、或问「新同事 XX 入职了，帮我准备一下」时使用。"
metadata:
  requires:
    bins: ["lark-cli"]
---

# 新人入职自动化工作流

**CRITICAL — 开始前 MUST 先用 Read 工具读取 [`lark-shared/SKILL.md`](https://github.com/larksuite/cli/blob/main/skills/lark-shared/SKILL.md)，其中包含认证、权限处理和安全规则。**

## 适用场景

- "新同事 XX 入职了" / "帮我准备新人入职流程"
- "XX 下周一入职，帮我提前安排好" / "启动入职流程"
- "新人 onboarding" / "准备入职材料"
- "把新同事加到项目群里" / "给新员工创建入职任务"

编排 **5 个业务域**（通讯录 + 即时通讯 + 任务 + 知识库 + 日历），实现入职流程一键自动化，是跨域编排能力的极致展示。

```
新人信息 ──► 自动化流程
              │
    ┌─────────┼─────────┬─────────┬─────────┐
    ▼         ▼         ▼         ▼         ▼
  Contact    IM       Task      Wiki     Calendar
  确认身份  拉入群聊  创建任务  发送必读  安排培训
```

## 前置条件

支持 **user 身份**。执行前确保已授权：

```bash
lark-cli auth login --domain contact,im,task,wiki,calendar
```

## 工作流

```
用户提供新人信息 ──► contact +search（确认身份）
                         │
                         ▼
                   确认执行计划
                         │
          ┌──────────────┼──────────────┐──────────────┐
          ▼              ▼              ▼              ▼
    im（拉入群聊     task（创建      wiki（发送     calendar
     + 发欢迎消息）  入职任务清单）  知识库必读）  （安排培训日程）
          │              │              │              │
          └──────────────┼──────────────┘──────────────┘
                         ▼
                    执行报告
```

### Step 1: 确认新人身份

参考 [`lark-contact/SKILL.md`](https://github.com/larksuite/cli/blob/main/skills/lark-contact/SKILL.md)。

```bash
# 搜索新人信息
lark-cli contact +search --query "<新人姓名>" --format json
```

从返回结果中获取：
- `user_id`：用于后续所有操作
- `name`：确认是否为目标新人
- `department`：部门信息（用于确定拉入哪些群）
- `email`：邮箱（备用联系方式）

> **多个匹配时**：列出供用户选择。
> **找不到时**：可能新人账号尚未创建，提示用户稍后重试或手动提供 user_id。

### Step 2: 收集入职配置

向用户确认以下信息（部分可按部门自动推断）：

```markdown
## 入职配置确认

- **新人**：{姓名}（{部门}）
- **入职日期**：{日期}

请确认以下操作（输入序号取消不需要的步骤）：
1. ✅ 拉入团队群聊：{推断的群名列表，或请用户指定}
2. ✅ 创建入职任务清单（默认 7 项）
3. ✅ 发送知识库必读文档
4. ✅ 安排 1v1 入职引导会议
5. ✅ 发送欢迎消息

如需修改，请告诉我。确认后输入"执行"。
```

> **⚠️ 安全规则：所有写入操作前必须展示计划并获得用户确认。**

### Step 3: 拉入群聊 + 发送欢迎消息

参考 [`lark-im/SKILL.md`](https://github.com/larksuite/cli/blob/main/skills/lark-im/SKILL.md)。

**方式 A：加入已有群聊（默认）**

```bash
# 搜索目标群聊
lark-cli im +chats-search --query "<群名关键词>" --format json

# 拉入群聊
lark-cli im chats members create --params '{"chat_id":"<chat_id>"}' \
  --data '{"id_list":["<user_id>"]}' --format json
```

> **注意**：拉人入群可能需要 bot 身份或群管理员权限。如失败，提示用户手动邀请。

**方式 B：创建专属入职群（用户要求时）**

> v1.0.4+ 支持 `--as user` 创建群聊，群主为当前用户而非 bot，更适合团队管理。

```bash
# 创建入职专属群（user 身份，群主为当前用户）
lark-cli im +chat-create --as user \
  --name "{新人姓名} 入职引导群" \
  --user-ids "<新人user_id>,<leader_user_id>" \
  --format json
```

**发送欢迎消息：**

```bash
# 在群里发送欢迎消息
lark-cli im +messages-send \
  --receive-id "<chat_id>" \
  --receive-id-type "chat_id" \
  --msg-type "text" \
  --content '{"text":"🎉 欢迎 @{新人姓名} 加入团队！\n\n{新人姓名}将加入{部门}，负责{岗位方向}。\n\n大家多多支持 🙌"}'
```

### Step 4: 创建入职任务清单

参考 [`lark-task/SKILL.md`](https://github.com/larksuite/cli/blob/main/skills/lark-task/SKILL.md)。

**默认入职任务（可定制）：**

```bash
# 逐个创建任务（串行执行）
lark-cli task +create --summary "完成飞书账号基础设置（头像、签名、部门）" \
  --due "<入职日+1天>" --assignee "<user_id>"

lark-cli task +create --summary "阅读团队知识库必读文档" \
  --due "<入职日+3天>" --assignee "<user_id>"

lark-cli task +create --summary "完成 IT 设备领取和权限申请" \
  --due "<入职日+1天>" --assignee "<user_id>"

lark-cli task +create --summary "与直属 Leader 完成 1v1 沟通" \
  --due "<入职日+3天>" --assignee "<user_id>"

lark-cli task +create --summary "了解团队当前项目和 OKR" \
  --due "<入职日+5天>" --assignee "<user_id>"

lark-cli task +create --summary "与核心协作方完成 Meet & Greet" \
  --due "<入职日+7天>" --assignee "<user_id>"

lark-cli task +create --summary "完成入职培训课程" \
  --due "<入职日+14天>" --assignee "<user_id>"
```

> **定制化**：如果用户指定了特定任务（如"加上完成代码仓库权限申请"），在默认列表中增加。

### Step 5: 发送知识库必读

参考 [`lark-wiki/SKILL.md`](https://github.com/larksuite/cli/blob/main/skills/lark-wiki/SKILL.md)。

```bash
# 列出知识空间
lark-cli wiki spaces list --format json

# 搜索新人必读文档
lark-cli wiki spaces nodes search --data '{"space_id":"<space_id>","query":"入职 新人 指南","page_size":10}' --format json
```

将搜索到的文档整理为必读清单，通过私聊消息发送给新人：

```bash
lark-cli im +messages-send \
  --receive-id "<user_id>" \
  --receive-id-type "user_id" \
  --msg-type "text" \
  --content '{"text":"Hi {新人姓名}，欢迎加入！🎉\n\n以下是你的入职必读文档：\n\n📚 必读清单：\n1. {文档1标题} - {链接}\n2. {文档2标题} - {链接}\n3. {文档3标题} - {链接}\n\n建议在入职前 3 天内阅读完毕。有问题随时找我！"}'
```

> **降级处理**：如果 wiki 域未授权或未找到入职文档，跳过此步，在报告中标注。

**创建入职 Wiki 页面（v1.0.7+，可选）：**

当用户希望为新人创建专属知识库页面时：

```bash
# 在知识库中创建新人专属页面（v1.0.7+ shortcut）
lark-cli wiki +node-create --space-id "<space_id>" \
  --parent-node-token "<parent_token>" \
  --title "{新人姓名} 入职指南"
```

> **提示**：`wiki +node-create` 替代了之前的原生 API 调用，更为简洁。

**将文档移动到入职目录（v1.0.10+，可选）：**

```bash
# 将已有文档移动到新人入职目录下（v1.0.10+ shortcut）
lark-cli wiki +move --space-id "<space_id>" \
  --node-token "<doc_node_token>" \
  --target-parent-token "<onboarding_folder_token>"
```

> **提示**：`wiki +move` 支持异步任务轮询，适合移动大型文档或目录。

**添加新人为 Wiki 空间成员（v1.0.10+，可选）：**

当知识库设有成员权限时，可直接将新人添加为空间成员：

```bash
# 查看 wiki members API 参数结构
lark-cli schema wiki.spaces.members.create

# 添加新人为 wiki 空间成员
lark-cli wiki spaces members create --params '{"space_id":"<space_id>"}' \
  --data '{"member_type":"userid","member_id":"<user_id>","member_role":"member"}' --format json
```

> **提示**：v1.0.10 的 lark-wiki skill 新增了 wiki 成员操作文档，确保新人入职后能直接访问团队知识库。

### Step 6: 安排入职培训日程

参考 [`lark-calendar/SKILL.md`](https://github.com/larksuite/cli/blob/main/skills/lark-calendar/SKILL.md)。

```bash
# 安排 1v1 入职引导会议（默认带视频会议链接，v1.0.8+）
lark-cli calendar +create \
  --summary "入职引导 - {新人姓名} & {Leader姓名}" \
  --start "<入职日>T10:00:00+08:00" \
  --end "<入职日>T11:00:00+08:00" \
  --attendees "<新人use_id>,<leader_user_id>"
```

> **注意**：v1.0.8+ `+create` 默认附带视频会议链接，新人即使不在办公室也可参与。
> **注意**：Leader 的 user_id 需要用户提供或通过 contact 搜索确认。

### Step 7: 输出执行报告

```markdown
## ✅ 入职流程执行报告

### 新人信息
- **姓名**：{姓名}
- **部门**：{部门}
- **入职日期**：{日期}

### 执行结果

| # | 操作 | 状态 | 详情 |
|---|------|------|------|
| 1 | 拉入群聊 | ✅ 已完成 | 已加入"XX团队群"、"XX项目群" |
| 2 | 欢迎消息 | ✅ 已发送 | 已在团队群发送欢迎消息 |
| 3 | 入职任务 | ✅ 已创建 | 7 项任务，截止日期从 {D+1} 到 {D+14} |
| 4 | 必读文档 | ✅ 已发送 | 3 篇文档链接已私聊发送 |
| 5 | 培训日程 | ✅ 已创建 | {入职日} 10:00-11:00 入职引导 |

### 后续建议
- Day 1：关注新人是否完成账号设置
- Day 3：检查必读文档阅读情况
- Day 7：确认 Meet & Greet 完成情况
- Day 14：入职培训完成确认
```

## 容错机制

| 异常场景 | 处理方式 |
|---------|---------|
| 新人账号未创建 | 提示稍后重试，或手动提供 user_id |
| 拉群权限不足 | 标注"需手动邀请"，不阻塞其余步骤 |
| 知识库无"入职/新人"相关文档 | 跳过发送必读，提示用户手动指定 |
| 日程创建失败 | 标注失败原因，建议手动创建 |
| 某一步骤失败 | 记录错误，继续执行后续步骤 |

## 权限表

| 步骤 | 命令 | 所需 scope | 是否必选 |
|------|------|-----------|---------|
| 搜索用户 | `contact +search` | `contact:user.base:readonly` | ✅ 必选 |
| 搜索群聊 | `im +chats-search` | `im:chat:readonly` | ✅ 必选 |
| 拉入群聊 | `im chats members create` | `im:chat:member:write` | ✅ 必选 |
| 创建群聊 | `im +chat-create --as user` | `im:chat:create_by_user` | 可选（方式 B） |
| 发送消息 | `im +messages-send` | `im:message` | ✅ 必选 |
| 创建任务 | `task +create` | `task:task:write` | ✅ 必选 |
| 搜索知识库 | `wiki spaces nodes search` | `wiki:wiki:readonly` | 可选 |
| 创建 Wiki 页面 | `wiki +node-create` | `wiki:wiki` | 可选（创建新人专属页） |
| 移动 Wiki 节点 | `wiki +move` | `wiki:wiki` | 可选（v1.0.10+） |
| 添加 Wiki 成员 | `wiki spaces members create` | `wiki:wiki:member:create` | 可选（v1.0.10+） |
| 创建日程 | `calendar +create` | `calendar:calendar.event:write` | 可选 |

## 参考

- [lark-shared](https://github.com/larksuite/cli/blob/main/skills/lark-shared/SKILL.md) — 认证、权限（必读）
- [lark-contact](https://github.com/larksuite/cli/blob/main/skills/lark-contact/SKILL.md) — `+search` 详细用法
- [lark-im](https://github.com/larksuite/cli/blob/main/skills/lark-im/SKILL.md) — 群聊管理、消息发送
- [lark-task](https://github.com/larksuite/cli/blob/main/skills/lark-task/SKILL.md) — 任务创建
- [lark-wiki](https://github.com/larksuite/cli/blob/main/skills/lark-wiki/SKILL.md) — 知识库搜索
- [lark-calendar](https://github.com/larksuite/cli/blob/main/skills/lark-calendar/SKILL.md) — `+create`
