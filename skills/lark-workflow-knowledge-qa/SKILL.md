---
name: lark-workflow-knowledge-qa
version: 1.0.0
description: >
  知识库问答：在飞书知识库中搜索相关文档，读取内容后进行 RAG 式问答，返回精准回答并附上来源引用链接。
  当用户需要查阅公司文档、询问政策/流程/规范、或问「知识库里关于 XX 怎么说的」时使用。
metadata:
  requires:
    bins:
      - lark-cli
---

# 知识库问答工作流

**CRITICAL — 开始前 MUST 先用 Read 工具读取 [`lark-shared/SKILL.md`](https://github.com/larksuite/cli/blob/main/skills/lark-shared/SKILL.md)，其中包含认证、权限处理和安全规则。**

## 适用场景

- "知识库里关于 XX 怎么说的" / "公司的 YY 政策是什么"
- "报销流程是什么" / "请假需要提前几天"
- "XX 技术方案文档在哪里" / "找一下关于 YY 的文档"
- "帮我从知识库查一下 ZZ" / "文档里有没有关于 XX 的说明"
- "知识库搜索 XX" / "Knowledge base Q&A"

## 核心价值

实现飞书知识库的 **RAG（检索增强生成）式问答**：先搜索定位相关文档，再读取内容，最后基于真实文档内容回答问题并附上来源引用。

```
用户问题 ──► 搜索知识库 ──► 定位相关文档 ──► 读取内容 ──► AI 问答 ──► 回答 + 引用
```

## 前置条件

支持 **user 身份**。执行前确保已授权：

```bash
lark-cli auth login --domain wiki,doc,drive
```

## 工作流

```
用户问题（自然语言）
        │
        ├─► wiki（搜索知识空间节点）──► 匹配文档列表
        │
        ├─► drive files search（补充搜索）──► 匹配文档列表（可选）
        │
        ▼
   合并去重 → 按相关性排序 → 取 Top N
        │
        ▼
   逐个 docs +fetch（读取文档内容）
        │
        ▼
   AI 综合分析（问题 + 文档内容）
        │
        ▼
   结构化回答 + 来源引用链接
```

### Step 1: 解析用户问题

从用户的自然语言问题中提取关键搜索词：

- "报销流程是什么" → 搜索词："报销流程"
- "关于 API 接入的规范" → 搜索词："API 接入 规范"
- "新员工入职需要准备什么" → 搜索词："新员工 入职"

> **注意**：搜索词应去除停用词，保留核心名词和动词，2-4 个词为佳。

### Step 2: 搜索知识库

参考 [`lark-wiki/SKILL.md`](https://github.com/larksuite/cli/blob/main/skills/lark-wiki/SKILL.md)。

**方式 A — 知识库节点搜索：**

```bash
# 列出可用的知识空间
lark-cli wiki spaces list --format json

# 在知识空间中搜索
lark-cli wiki spaces nodes search --data '{"space_id":"<space_id>","query":"<搜索词>","page_size":10}' --format json
```

**方式 B — 云空间文件搜索（补充）：**

参考 [`lark-drive/SKILL.md`](https://github.com/larksuite/cli/blob/main/skills/lark-drive/SKILL.md)。

```bash
# 全局搜索文档
lark-cli drive files search --data '{"search_key":"<搜索词>","count":10,"doc_types":["docx","doc","wiki"]}' --format json
```

### Step 3: 筛选与排序

从搜索结果中筛选最相关的文档：

1. 合并两种搜索的结果，按 `doc_token` 去重
2. 优先选择标题中包含搜索词的文档
3. 取 **Top 3-5** 个最相关的文档进入下一步

> **上下文管理**：最多读取 5 篇文档。如果结果过多，提示用户缩小搜索范围。

### Step 4: 读取文档内容

参考 [`lark-doc/SKILL.md`](https://github.com/larksuite/cli/blob/main/skills/lark-doc/SKILL.md)。

对于每篇筛选出的文档：

```bash
# Wiki 链接需先解析真实 token
lark-cli wiki spaces get_node --params '{"token":"<wiki_token>"}'

# 读取文档内容
lark-cli docs +fetch --doc "<doc_token>" --format markdown
```

> **长文档处理**：如果单篇文档过长（>20KB），仅读取与问题相关的章节。可通过文档的大纲（# 标题层级）定位相关段落。

### Step 5: AI 综合问答

基于用户问题 + 读取到的文档内容，生成回答：

```markdown
## 💡 回答：{用户问题的简短回述}

{基于文档内容生成的详细回答}

{如果有具体步骤、流程，使用有序列表}

### 📚 来源文档

| # | 文档标题 | 相关程度 | 链接 |
|---|---------|---------|------|
| 1 | {文档标题 1} | ⭐⭐⭐ 高度相关 | [查看文档]({链接}) |
| 2 | {文档标题 2} | ⭐⭐ 部分相关 | [查看文档]({链接}) |

### 💬 补充说明
- {如果文档中信息有冲突，指出冲突}
- {如果信息可能已过时，提示最后更新时间}
- {如果问题未在文档中完全覆盖，说明缺失部分}
```

**生成规则：**

1. **回答必须基于文档内容**：不能凭空生成，每个要点应能追溯到具体文档
2. **引用标注**：在回答中用 `[1]` `[2]` 标注信息来源
3. **诚实声明**：如果知识库中没有找到相关内容，明确告知用户"未在知识库中找到关于 XX 的内容"
4. **时效提示**：如果文档创建时间或编辑时间较久（>6 个月），提示用户信息可能已过时
5. **多文档综合**：如果多篇文档涉及同一主题，综合所有信息给出完整回答

### Step 6: 追问与深入（交互式）

回答后，主动提供追问建议：

```markdown
### 🔍 您可能还想了解
- "{相关问题 1}"
- "{相关问题 2}"
- "查看完整文档：[{标题}]({链接})"
```

## 容错机制

| 异常场景 | 处理方式 |
|---------|---------|
| 未找到相关文档 | 明确告知"未找到"，建议扩大搜索范围或换个关键词 |
| 文档无读取权限 | 列出标题但标注"无权限"，建议用户联系文档管理员 |
| 搜索结果过多 | 取 Top 5，提示用户缩小范围 |
| 文档内容过长 | 按章节标题定位相关段落，不读取全文 |
| wiki 域未授权 | 降级为仅使用 drive files search |
| 文档可能过时 | 在回答中标注文档最后编辑时间 |

## 权限表

| 步骤 | 命令 | 所需 scope | 是否必选 |
|------|------|-----------|---------|
| 知识库搜索 | `wiki spaces nodes search` | `wiki:wiki:readonly` | ✅ 必选 |
| Wiki 节点解析 | `wiki spaces get_node` | `wiki:wiki:readonly` | ✅ 必选 |
| 文档搜索 | `drive files search` | `drive:drive:readonly` | 可选（补充搜索） |
| 读取文档 | `docs +fetch` | `docx:document:readonly` | ✅ 必选 |

## 参考

- [lark-shared](https://github.com/larksuite/cli/blob/main/skills/lark-shared/SKILL.md) — 认证、权限（必读）
- [lark-wiki](https://github.com/larksuite/cli/blob/main/skills/lark-wiki/SKILL.md) — 知识库搜索与节点查询
- [lark-doc](https://github.com/larksuite/cli/blob/main/skills/lark-doc/SKILL.md) — `+fetch` 详细用法
- [lark-drive](https://github.com/larksuite/cli/blob/main/skills/lark-drive/SKILL.md) — 文件搜索详细用法
