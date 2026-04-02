---
name: lark-workflow-doc-summarizer
version: 1.0.0
description: >
  文档智能摘要：读取飞书云文档或知识库文档内容，生成结构化摘要（要点、关键决策、待确认事项）。
  当用户需要总结文档、提取文档要点、快速了解一篇长文档的核心内容、或问「帮我总结这个文档」时使用。
metadata:
  requires:
    bins:
      - lark-cli
---

# 文档智能摘要工作流

**CRITICAL — 开始前 MUST 先用 Read 工具读取 [`lark-shared/SKILL.md`](https://github.com/larksuite/cli/blob/main/skills/lark-shared/SKILL.md)，其中包含认证、权限处理和安全规则。**

## 适用场景

- "帮我总结这个文档" / "这篇文档讲了什么"
- "提取文档要点" / "这个文档的核心内容是什么"
- "帮我快速了解这篇文档" / "文档摘要"
- "总结一下这个链接里的内容" / "摘出关键决策"
- "批量摘要这几个文档" / "帮我把文档要点整理出来"

## 前置条件

支持 **user 身份**。执行前确保已授权：

```bash
lark-cli auth login --domain doc,drive,wiki
```

## 工作流

```
用户指定文档 ──► 解析输入类型
                    │
         ┌──────────┼──────────┐
         ▼          ▼          ▼
     /docs/ 链接  /wiki/ 链接  doc_token
         │          │          │
         │     wiki get_node   │
         │     提取 obj_token  │
         │          │          │
         └──────────┼──────────┘
                    ▼
            docs +fetch（Markdown）
                    │
                    ▼
              AI 分段摘要
                    │
            ┌───────┼───────┐
            ▼       ▼       ▼
          要点   关键决策  待确认项
                    │
                    ▼
         结构化摘要输出（可选：写回文档）
```

### Step 1: 解析文档来源

用户可能提供以下几种输入：

**A. 飞书文档链接（`/docs/` 或 `/docx/`）：**
```
https://xxx.feishu.cn/docs/<doc_token>
https://xxx.feishu.cn/docx/<doc_token>
```
直接提取 `doc_token`。

**B. 知识库链接（`/wiki/`）：**
```
https://xxx.feishu.cn/wiki/<wiki_token>
```

> **⚠️ 重要**：知识库链接的 token 不能直接用作 doc_token，必须先查询真实类型。

```bash
# 查询 wiki 节点信息
lark-cli wiki spaces get_node --params '{"token":"<wiki_token>"}'
```

从返回结果中提取：
- `node.obj_type`：文档类型（docx/doc/sheet/bitable 等）
- `node.obj_token`：真实文档 token
- `node.title`：文档标题

仅当 `obj_type` 为 `docx` 或 `doc` 时继续摘要。其他类型提示用户："该链接指向的是{类型}，暂不支持摘要。"

**C. 直接提供 doc_token：**

直接进入 Step 2。

### Step 2: 读取文档内容

参考 [`lark-doc/SKILL.md`](https://github.com/larksuite/cli/blob/main/skills/lark-doc/SKILL.md)。

```bash
# 读取文档内容（Markdown 格式）
lark-cli docs +fetch --doc "<doc_token>" --format markdown
```

> **长文档处理**：如果文档内容超过 AI 上下文限制（通常 > 50KB），采取以下策略：
> 1. 提示用户文档较长，将分段处理
> 2. 按大标题（`#` / `##`）拆分为段落
> 3. 逐段生成摘要，最后合并

### Step 3: AI 生成结构化摘要

将文档内容分析后，按以下结构输出：

```markdown
## 📄 文档摘要：{文档标题}

### 📌 核心要点
1. **{要点 1}** — 一句话说明
2. **{要点 2}** — 一句话说明
3. **{要点 3}** — 一句话说明

### 🔑 关键决策
- {决策 1}：选择了方案 X，原因是...
- {决策 2}：确定了时间节点为...

### ❓ 待确认 / 开放问题
- {问题 1}：文中提到但未给出结论
- {问题 2}：需要进一步讨论

### 📊 数据与指标（如有）
- {指标 1}：xxx
- {指标 2}：xxx

### 📝 一句话总结
> {不超过 50 字的核心总结}
```

**生成规则：**

1. **核心要点**：提取 3-7 个最重要的信息点，按重要性排序
2. **关键决策**：识别文档中的决策性内容（"决定"、"确定"、"采用"、"选择"等关键词）
3. **待确认项**：识别未决事项（"待定"、"需讨论"、"TBD"、"?"、"待确认"等）
4. **数据与指标**：如果文档包含数字、百分比、KPI 等，单独提取
5. **一句话总结**：概括全文核心观点

### Step 4: 批量摘要模式（可选）

当用户需要批量摘要多个文档时：

```bash
# 方式 1：用户提供多个链接/token
# 串行处理每个文档（每个文档间隔 1 秒）

# 方式 2：搜索并摘要
lark-cli drive files search --data '{"search_key":"<关键词>","count":10}' --format json
# 然后对搜索结果中的文档逐个 fetch + 摘要
```

批量模式输出：
```markdown
## 批量文档摘要（共 N 篇）

### 1. {文档 1 标题}
> {一句话总结}
- 要点：...

### 2. {文档 2 标题}
> {一句话总结}
- 要点：...

### 📊 跨文档洞察
- {跨文档的共性发现}
- {矛盾或冲突点}
```

### Step 5: 写回文档（可选，用户要求时）

参考 [`lark-doc/SKILL.md`](https://github.com/larksuite/cli/blob/main/skills/lark-doc/SKILL.md)。

```bash
# 在文档开头插入摘要
lark-cli docs +update --doc "<doc_token>" --mode prepend --markdown "## 📄 文档摘要\n\n<摘要内容>\n\n---\n\n"

# 或创建单独的摘要文档
lark-cli docs +create --title "摘要：{原文档标题}" --markdown "<摘要内容>"
```

## 容错机制

| 异常场景 | 处理方式 |
|---------|---------|
| 文档无权限 | 提示用户检查权限，或联系文档管理员 |
| wiki 链接指向非文档类型 | 提示"该链接指向的是{类型}，暂不支持摘要" |
| 文档内容为空 | 提示"该文档暂无内容" |
| 文档过长（>50KB） | 分段处理 + 合并摘要 |
| 批量处理中某个文档失败 | 记录失败，继续处理其余文档 |

## 权限表

| 步骤 | 命令 | 所需 scope | 是否必选 |
|------|------|-----------|---------|
| 读取文档 | `docs +fetch` | `docx:document:readonly` | ✅ 必选 |
| 解析 wiki | `wiki spaces get_node` | `wiki:wiki:readonly` | 可选（wiki 链接时） |
| 搜索文档 | `drive files search` | `drive:drive:readonly` | 可选（批量模式） |
| 写回文档 | `docs +update` / `docs +create` | `docx:document:write` | 可选（写回时） |

## 参考

- [lark-shared](https://github.com/larksuite/cli/blob/main/skills/lark-shared/SKILL.md) — 认证、权限（必读）
- [lark-doc](https://github.com/larksuite/cli/blob/main/skills/lark-doc/SKILL.md) — `+fetch`、`+create`、`+update` 详细用法
- [lark-wiki](https://github.com/larksuite/cli/blob/main/skills/lark-wiki/SKILL.md) — wiki 节点查询详细用法
- [lark-drive](https://github.com/larksuite/cli/blob/main/skills/lark-drive/SKILL.md) — 文件搜索详细用法
