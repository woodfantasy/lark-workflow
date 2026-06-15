---
name: lark-workflow-action-extractor
version: 1.5.0
description: "会议任务闭环：从会议纪要中提取行动项，自动创建飞书任务并通知责任人， 形成「会议→纪要→行动项→任务→通知」的完整闭环。当用户需要把会议纪要变成任务、 提取 action items、跟进会议决策、或问「把今天的会议纪要整理成任务」时使用。"
metadata:
  requires:
    bins: ["lark-cli"]
---

# 会议任务闭环工作流

**CRITICAL — 开始前 MUST 先用 Read 工具读取 [`lark-shared/SKILL.md`](https://github.com/larksuite/cli/blob/main/skills/lark-shared/SKILL.md)，其中包含认证、权限处理和安全规则。**

## 适用场景

- "把今天的会议纪要变成任务" / "从会议记录提取 action items"
- "帮我跟踪会议决策" / "会议行动项提取"
- "今天的会议有哪些待办" / "把会议结论变成任务分配下去"
- "Extract action items from meeting" / "meeting to tasks"
- "XX 会议的行动项是什么" / "帮我把会议里说的那些事情建成任务"

形成 **会议 → 纪要 → 行动项 → 任务 → 通知 → 跟踪** 的完整闭环。这是现有官方 Skill 都没有覆盖的端到端场景。

```
会议结束                  行动项提取                  任务创建 & 通知
───────── ► vc/minutes ► AI 分析提取 ► 用户确认 ► task + im ─────►
                │                          │                    │
                ▼                          ▼                    ▼
         纪要已存在                  展示行动项清单          责任人收到通知
```

## 前置条件

仅支持 **user 身份**。执行前确保已授权：

```bash
lark-cli auth login --domain vc,drive,contact,task,im
```

## 工作流

```
用户指定会议 ──► vc +search      ──► 定位会议 (meeting_id)
                    │
                    ▼
               vc +notes          ──► 获取纪要 doc_token
                    │
                    ▼
            ┌───────┴───────┐
            ▼               ▼
     docs +fetch       minutes 产物
    (纪要 Markdown)    (AI 待办/总结)
            │               │
            └───────┬───────┘
                    ▼
              AI 提取行动项
           {who, what, when}
                    │
                    ▼
             展示给用户确认 ◄── MUST：用户确认后才执行创建
                    │
                    ▼
          ┌─────────┴─────────┐
          ▼                   ▼
    task（批量创建）    im +messages-send
     设置截止日期          通知责任人
     分配责任人
```

### Step 1: 定位目标会议

**方式 A — 用户指定关键词搜索：**

参考 [`lark-vc/SKILL.md`](https://github.com/larksuite/cli/blob/main/skills/lark-vc/SKILL.md)。

```bash
# 搜索今天的会议
lark-cli vc +search --start "<today>" --end "<today>" --format json --page-size 30
```

如果返回多个会议，列出供用户选择：
```
找到 3 场今日会议：
1. 09:00 产品需求评审（meeting_id: xxx）
2. 14:00 技术方案讨论（meeting_id: yyy）
3. 16:00 周五复盘（meeting_id: zzz）

请问要从哪场会议提取行动项？（输入序号或"全部"）
```

**方式 B — 用户直接提供会议 ID 或链接：**

直接使用提供的 meeting_id 进入 Step 2。

### Step 2: 获取会议纪要

```bash
# 获取会议关联的纪要信息
lark-cli vc +notes --meeting-ids "<meeting_id>"
```

从返回结果中提取：
- `note_doc_token`：纪要文档 Token
- `verbatim_doc_token`：逐字稿文档 Token（如有）

> **无纪要时的处理**：如果会议没有纪要（`no notes available`），提示用户：
> - "该会议暂无纪要。您可以口述会议结论，我来整理成行动项。"
> - 或尝试从妙记中获取 AI 产物（Step 3B）

**补充路径（v1.0.7+）：** 当 `vc +notes` 未返回结果时，可通过日历事件关联 API 提取纪要 doc_token：

```bash
# 通过 VC 的日历关联 API 获取 note doc token（v1.0.7+ 增强）
lark-cli vc +notes --meeting-ids "<meeting_id>"
```

> **提示**：v1.0.7 增强了从日历事件关联 API 提取 note doc token 的能力，覆盖了更多会议场景。

### Step 3A: 读取纪要正文

参考 [`lark-doc/SKILL.md`](https://github.com/larksuite/cli/blob/main/skills/lark-doc/SKILL.md)。

```bash
# 先获取纪要文档的访问链接
lark-cli drive metas batch_query --data '{"request_docs": [{"doc_type": "docx", "doc_token": "<note_doc_token>"}], "with_url": true}'

# 读取纪要文档内容（Markdown 格式）
lark-cli docs +fetch --api-version v2 --doc "<note_doc_token>" --doc-format markdown
```

### Step 3B: 获取妙记 AI 产物（补充数据源）

参考 [`lark-minutes/SKILL.md`](https://github.com/larksuite/cli/blob/main/skills/lark-minutes/SKILL.md)、[`lark-vc/SKILL.md`](https://github.com/larksuite/cli/blob/main/skills/lark-vc/SKILL.md) 和 [`lark-vc/SKILL.md`](https://github.com/larksuite/cli/blob/main/skills/lark-vc/SKILL.md)。

**路径 A — 通过 minutes +get 获取 AI 产物（原有路径）：**

```bash
# 首先获取会议的妙记 Token (需要 v1.0.6+ 支持)
lark-cli vc +recording --meeting-id "<meeting_id>" --format json

# 获取妙记的 AI 产物（总结、待办、章节）
lark-cli minutes +get --minute-token "<minute_token>" --format json
```

**路径 B — 通过 note domain 获取逐字稿（v1.0.53+ 推荐）：**

```bash
# 获取 note 元数据（包含 display_type 和文档 tokens）
lark-cli note +detail --note-id "<note_id>" --format json

# 直接获取统一逐字稿（Markdown 格式，支持中英日）
lark-cli note +transcript --note-id "<note_id>" --transcript-format markdown
```

> **使用建议**：
> - `note +transcript` 提供完整的逐字稿文本，适合从中提取细粒度行动项
> - `minutes +get` 提供 AI 生成的待办/总结，适合快速获取结构化结论
> - 两个路径互为补充，优先使用 `note +transcript`（更完整），再用 `minutes +get` 获取 AI 待办作为补充

> **note_id 来源**：`vc +notes` 返回结果中的 `note_id` 字段（v1.0.53+ 可用）

> **注意**：妙记的待办提取（`todo` 字段）是很好的补充数据源。AI 应将妙记待办与纪要正文中的行动项合并去重。

### Step 4: AI 提取行动项

将纪要正文 + 妙记待办合并分析，提取行动项。

**提取规则：**

1. **识别关键词**：
   - 责任人线索："负责"、"跟进"、"由 XX"、"XX 来做"、"分配给"、"@XX"
   - 行动线索："需要"、"完成"、"提交"、"确认"、"修改"、"准备"、"调研"、"输出"
   - 时间线索："本周"、"下周"、"月底前"、"by Friday"、"deadline"、"截止"

2. **结构化格式**：
   ```
   {
     "who": "责任人姓名",
     "what": "具体行动项描述",
     "when": "截止时间（如能提取）",
     "source": "来源（纪要正文 / 妙记待办）",
     "priority": "优先级推断（高/中/低）"
   }
   ```

3. **人名解析**：
   - 提取到的人名使用 `contact` 搜索验证
   ```bash
   lark-cli contact +search-user --query "<姓名>" --format json
   ```
   - 匹配成功：记录 `user_id`，用于后续任务分配
   - 匹配失败：标注"待确认"，不阻塞流程

### Step 5: 展示行动项供用户确认

> **⚠️ 安全规则：在创建任务前 MUST 展示行动项清单，等待用户确认。**

展示格式：

```markdown
## 从「{会议名称}」提取的行动项

| # | 责任人 | 行动项 | 截止日期 | 优先级 | 来源 |
|---|--------|--------|---------|--------|------|
| 1 | 张三 ✅ | 完成 API 接口文档 | 下周三 | 高 | 纪要 |
| 2 | 李四 ✅ | 调研竞品方案 | 本周五 | 中 | 妙记 |
| 3 | 王五 ⚠️ | 准备测试环境 | 未指定 | 低 | 纪要 |

✅ = 已匹配到飞书用户    ⚠️ = 未匹配，任务将创建但不指定负责人

确认创建以上任务？（输入"确认"执行，或修改后再确认）
```

用户可以：
- 确认全部创建
- 删除某些行动项（"去掉第 3 项"）
- 修改责任人或截止日期
- 补充遗漏的行动项

### Step 6: 批量创建任务

参考 [`lark-task/SKILL.md`](https://github.com/larksuite/cli/blob/main/skills/lark-task/SKILL.md)。

**6a. 去重检查（推荐，v1.0.11+）：**

在创建任务前，搜索是否已有相似任务，避免重复创建：

```bash
# 搜索是否已有相似任务
lark-cli task +search --query "<行动项关键词>" --completed=false --format json
```

> **提示**：如果搜索到与行动项高度相似的已有任务，在确认界面标注"⚡ 疑似已有任务"，让用户决定是否跳过。

**6b. 创建任务：**

```bash
# 逐个创建任务（串行执行，避免并发冲突）
lark-cli task +create --summary "<行动项描述>" --due "<截止时间 ISO 8601>" --assignee "<user_id>"
```

**6c. 加入任务清单（可选，v1.0.10+）：**

如果用户指定了目标任务清单，可将新创建的任务批量加入：

```bash
# 将任务加入指定清单的特定分组（v1.0.10+ 支持 --section-guid）
lark-cli task +tasklist-task-add --tasklist-id "<tasklist_guid>" \
  --task-id "<task_guid_1>,<task_guid_2>" \
  --section-guid "<section_guid>"
```

> **提示**：`--section-guid` 可将行动项精确分配到任务清单的特定自定义分组（如"会议行动项"分组），便于后续跟踪。

**创建规则：**
1. 串行创建，每个任务间隔 0.5~1 秒
2. 截止日期使用 ISO 8601 格式
3. 如果未匹配到 user_id，创建任务但不指定负责人
4. 记录每个任务的创建结果（task_id + 成功/失败）

> **注意**：`+create` 命令的参数以 `lark-task/SKILL.md` 中的 reference 文档为准。执行前先读取对应的 reference，确认参数名和格式。

### Step 7: 通知责任人

参考 [`lark-im/SKILL.md`](https://github.com/larksuite/cli/blob/main/skills/lark-im/SKILL.md)。

```bash
# 向每位责任人发送通知（通过 bot 身份或 user 身份）
lark-cli im +messages-send --receive-id "<user_id>" --receive-id-type "user_id" --msg-type "text" --content '{"text":"你好，来自「<会议名称>」的行动项已创建为任务：\n\n<行动项描述>\n截止日期：<截止时间>\n\n请及时处理 🙏"}'
```

> **注意**：发送消息可能需要 bot 身份。如果 user 身份无法发送，提示用户使用 `--as bot`。

### Step 8: 输出执行报告

```markdown
## 执行报告

### 会议信息
- 会议名称：{会议名称}
- 会议时间：{时间}
- 纪要链接：[查看纪要]({链接})

### 任务创建结果
| # | 行动项 | 责任人 | 截止 | 任务状态 |
|---|--------|--------|------|---------|
| 1 | 完成 API 接口文档 | 张三 | 下周三 | ✅ 已创建 |
| 2 | 调研竞品方案 | 李四 | 本周五 | ✅ 已创建 |
| 3 | 准备测试环境 | 王五 | — | ✅ 已创建（未指定负责人）|

### 通知发送结果
- 张三：✅ 已通知
- 李四：✅ 已通知
- 王五：⚠️ 未匹配用户，跳过通知

### 后续建议
- 您可以使用 `lark-cli task +get-my-tasks` 查看任务进度
- 下次会议前可以回顾任务完成情况
```

## 容错机制

| 异常场景 | 处理方式 |
|---------|---------|
| 会议无纪要 | 提示用户口述结论，手动提取行动项 |
| 纪要正文为空 | 尝试从妙记 AI 产物中获取待办 |
| 人名匹配失败 | 标注"待确认"，创建任务但不分配 |
| 任务创建失败 | 记录错误，继续创建其余任务，最终报告失败项 |
| 消息发送失败 | 记录错误，不阻塞流程，建议用户手动通知 |
| 无截止时间 | 创建任务但不设置截止日期，在报告中标注 |

## 权限表

| 步骤 | 命令 | 所需 scope | 是否必选 |
|------|------|-----------|---------|
| 搜索会议 | `vc +search` | `vc:meeting:readonly` | ✅ 必选 |
| 获取纪要 | `vc +notes` | `vc:meeting:readonly` | ✅ 必选 |
| 读取纪要正文 | `docs +fetch --api-version v2` | `docx:document:readonly` | ✅ 必选 |
| 获取妙记产物 | `minutes +get` | `minutes:minute:readonly` | 可选 |
| 获取逐字稿（新） | `note +detail` / `note +transcript` | `minutes:minute:readonly` | 可选（v1.0.53+） |
| 文档元数据 | `drive metas batch_query` | `drive:drive:readonly` | 可选 |
| 搜索用户 | `contact +search-user` | `contact:user.base:readonly` | 推荐 |
| 搜索任务(去重) | `task +search` | `task:task:read` | 推荐（v1.0.11+） |
| 创建任务 | `task +create` | `task:task:write` | ✅ 必选 |
| 加入清单 | `task +tasklist-task-add` | `task:tasklist:write` | 可选（v1.0.10+） |
| 发送通知 | `im +messages-send` | `im:message:send_as_bot` 或 `im:message` | 推荐 |

## 参考

- [lark-shared](https://github.com/larksuite/cli/blob/main/skills/lark-shared/SKILL.md) — 认证、权限（必读）
- [lark-vc](https://github.com/larksuite/cli/blob/main/skills/lark-vc/SKILL.md) — `+search`、`+notes` 详细用法
- [lark-minutes](https://github.com/larksuite/cli/blob/main/skills/lark-minutes/SKILL.md) — 妙记产物详细用法
- [lark-doc](https://github.com/larksuite/cli/blob/main/skills/lark-doc/SKILL.md) — `+fetch` 详细用法
- [lark-contact](https://github.com/larksuite/cli/blob/main/skills/lark-contact/SKILL.md) — `+search` 详细用法
- [lark-task](https://github.com/larksuite/cli/blob/main/skills/lark-task/SKILL.md) — 任务创建详细用法
- [lark-im](https://github.com/larksuite/cli/blob/main/skills/lark-im/SKILL.md) — 消息发送详细用法
