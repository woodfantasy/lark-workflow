---
name: lark-workflow-wiki-auditor
version: 1.1.0
description: "知识库健康度审计：遍历飞书知识空间的文档节点，分析最后更新时间和结构层级， 识别过时文档、孤立节点和结构问题，生成知识库健康度报告。 当用户需要检查知识库状态、或问「哪些文档过时了」时使用。"
metadata:
  requires:
    bins: ["lark-cli"]
---

# 知识库健康度审计工作流

**CRITICAL — 开始前 MUST 先用 Read 工具读取 [`lark-shared/SKILL.md`](https://github.com/larksuite/cli/blob/main/skills/lark-shared/SKILL.md)，其中包含认证、权限处理和安全规则。**

## 适用场景

- "检查一下知识库的状态" / "哪些文档过时了"
- "知识库健康度报告" / "审计一下知识库"
- "有没有很久没更新的文档" / "知识库结构分析"
- "知识库优化建议" / "Wiki audit"
- "清理一下知识库" / "哪些文档可能需要更新了"

与已有 Skill 形成「知识管理三件套」：
- `knowledge-qa`：**搜索问答** — 使用知识库
- `doc-summarizer`：**文档摘要** — 理解知识库
- **`wiki-auditor`**（本 Skill）：**健康审计** — 维护知识库

## 前置条件

支持 **user 身份**。执行前确保已授权：

```bash
lark-cli auth login --domain wiki,drive
```

## 工作流

```
指定知识空间 ──► wiki spaces list（列出可用空间）
                       │
                       ▼
              wiki nodes（遍历节点树）
                       │
                       ▼
              drive metas（获取文档元数据）
                       │
                       ▼
              AI 健康度分析
              ┌────────┼────────┐
              ▼        ▼        ▼
           过时检测  结构分析  覆盖评估
                       │
                       ▼
              健康度审计报告
```

### Step 1: 列出知识空间

参考 [`lark-wiki/SKILL.md`](https://github.com/larksuite/cli/blob/main/skills/lark-wiki/SKILL.md)。

```bash
# 列出所有可见的知识空间
lark-cli wiki spaces list --format json
```

如果有多个空间，列出供用户选择：
```markdown
找到 3 个知识空间：
1. 产品团队知识库（space_id: xxx）
2. 技术文档库（space_id: yyy）
3. 公司公共知识库（space_id: zzz）

请选择要审计的知识空间（输入序号，或"全部"）
```

### Step 2: 遍历知识节点

```bash
# 获取空间根节点下的子节点
lark-cli wiki spaces nodes list --params '{"space_id":"<space_id>","page_size":50}' --format json

# 获取子节点的子节点（递归）
lark-cli wiki spaces nodes list --params '{"space_id":"<space_id>","parent_node_token":"<parent_token>","page_size":50}' --format json
```

对每个节点记录：
- `node_token`：节点 token
- `obj_token`：关联文档 token
- `obj_type`：文档类型
- `title`：标题
- `has_child`：是否有子节点
- 节点层级深度（根据递归深度计算）

> **⚠️ 数据量控制**：大型知识库可能有数百个节点。建议限制最大遍历深度为 3-4 层，超过时提示用户缩小范围。

### Step 3: 获取文档元数据

参考 [`lark-drive/SKILL.md`](https://github.com/larksuite/cli/blob/main/skills/lark-drive/SKILL.md)。

```bash
# 批量获取文档元数据（每次最多 15 个）
lark-cli drive metas batch_query --data '{
  "request_docs": [
    {"doc_type": "docx", "doc_token": "<token1>"},
    {"doc_type": "docx", "doc_token": "<token2>"}
  ],
  "with_url": true
}' --format json
```

从元数据中提取：
- `latest_modify_time`：最后修改时间
- `create_time`：创建时间
- `owner`：文档拥有者
- `url`：文档链接

### Step 4: AI 健康度分析

**4.1 过时文档检测：**

| 过时等级 | 最后更新距今 | 标记 |
|---------|-----------|------|
| 新鲜 | < 30 天 | 🟢 |
| 正常 | 30-90 天 | 🟡 |
| 陈旧 | 90-180 天 | 🟠 |
| 过时 | > 180 天 | 🔴 |

**4.2 结构分析：**
- 层级深度分布（过深 > 4 层不利于发现）
- 孤立节点（有文档但无子节点也无兄弟节点的叶子节点比例）
- 空节点（有节点但关联的文档为空或已删除）
- 目录均衡度（某个目录下文档过多 > 20 篇）

**4.3 覆盖评估：**
- 总文档数
- 按文档类型分布（docx / sheet / bitable 等）
- 按创建者分布（是否高度集中于少数人）

### Step 5: 生成审计报告

```markdown
## 📚 知识库健康度报告

### 空间信息
- **知识空间**：{空间名称}
- **审计时间**：{当前日期}
- **总文档数**：{N} 篇
- **节点层级**：最深 {M} 层

### 🏥 健康度评分

| 维度 | 评分 | 说明 |
|------|------|------|
| 内容新鲜度 | {X}/100 | 基于文档更新时间分布 |
| 结构合理性 | {Y}/100 | 基于层级深度和目录均衡度 |
| 覆盖完整性 | {Z}/100 | 基于文档数量和类型分布 |
| **综合评分** | **{总分}/100** | **{优秀/良好/需改进/较差}** |

### 🔴 过时文档清单（需关注）
| # | 文档标题 | 最后更新 | 距今 | 拥有者 | 链接 |
|---|---------|---------|------|--------|------|
| 1 | XX 接口文档 v1.0 | 2025-06-15 | 292天 | 张三 | [查看]({链接}) |
| 2 | YY 项目设计方案 | 2025-08-20 | 226天 | 李四 | [查看]({链接}) |
| 3 | ZZ 操作手册 | 2025-09-10 | 205天 | 王五 | [查看]({链接}) |

### 📊 新鲜度分布
```
🟢 新鲜（<30天）   ████████░░  35%  ({N}篇)
🟡 正常（30-90天）  ██████░░░░  28%  ({N}篇)
🟠 陈旧（90-180天） ████░░░░░░  22%  ({N}篇)
🔴 过时（>180天）   ███░░░░░░░  15%  ({N}篇)
```

### 🗂️ 结构分析
- **层级分布**：1层 {N}个 | 2层 {N}个 | 3层 {N}个 | 4层+ {N}个
- ⚠️ 「{目录名}」下有 {N} 篇文档，建议拆分子目录
- ⚠️ 发现 {N} 个空节点（可能是已删除的文档引用）
- ⚠️ 层级深度 > 4 层的分支有 {N} 个

### 👤 贡献者分布
| 作者 | 文档数 | 占比 |
|------|--------|------|
| 张三 | 25 | 35% |
| 李四 | 18 | 25% |
| 其他 | 28 | 40% |

### 💡 优化建议
1. **{建议 1}**：{针对过时文档的具体建议}
2. **{建议 2}**：{针对结构问题的具体建议}
3. **{建议 3}**：{针对贡献者集中度的建议}
```

### Step 6: 自动创建修复建议 Wiki（可选，v1.0.7+）

当用户希望将审计结果写入知识库时：

```bash
# 在知识库中创建审计报告节点（v1.0.7+ shortcut）
lark-cli wiki +node-create --space-id "<space_id>" \
  --parent-node-token "<parent_token>" \
  --title "知识库审计报告 (<date>)"
```

> **提示**：审计报告可直接嵌入知识库，方便团队查看和跟踪修复进度。

## 容错机制

| 异常场景 | 处理方式 |
|---------|---------|
| 知识空间无访问权限 | 提示用户加入空间或联系管理员 |
| 节点数量 > 200 | 限制遍历深度为 2 层，提示用户缩小范围 |
| 文档元数据获取失败 | 记录失败节点，其余正常分析 |
| 空间无文档 | 提示"该知识空间暂无文档" |
| drive 域未授权 | 无法获取修改时间，仅做结构分析 |

## 权限表

| 步骤 | 命令 | 所需 scope | 是否必选 |
|------|------|-----------|---------|
| 列出空间 | `wiki spaces list` | `wiki:wiki:readonly` | ✅ 必选 |
| 遍历节点 | `wiki spaces nodes list` | `wiki:wiki:readonly` | ✅ 必选 |
| 文档元数据 | `drive metas batch_query` | `drive:drive:readonly` | 推荐 |
| 创建审计报告 | `wiki +node-create` | `wiki:wiki` | 可选（写入审计结果时） |

## 参考

- [lark-shared](https://github.com/larksuite/cli/blob/main/skills/lark-shared/SKILL.md) — 认证、权限（必读）
- [lark-wiki](https://github.com/larksuite/cli/blob/main/skills/lark-wiki/SKILL.md) — 知识空间遍历详细用法
- [lark-drive](https://github.com/larksuite/cli/blob/main/skills/lark-drive/SKILL.md) — 文档元数据查询
