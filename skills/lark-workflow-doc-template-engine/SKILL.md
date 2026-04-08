---
name: lark-workflow-doc-template-engine
version: 1.0.0
description: "文档模板引擎：读取飞书文档模板，从多维表格或电子表格中获取数据源， 自动进行变量替换并批量生成文档。当用户需要基于模板批量生成文档、 或问「用 XX 模板帮我生成一批文档」时使用。"
metadata:
  requires:
    bins: ["lark-cli"]
---

# 文档模板引擎工作流

**CRITICAL — 开始前 MUST 先用 Read 工具读取 [`lark-shared/SKILL.md`](https://github.com/larksuite/cli/blob/main/skills/lark-shared/SKILL.md)，其中包含认证、权限处理和安全规则。**

## 适用场景

- "用 XX 模板创建文档" / "基于模板批量生成报告"
- "把多维表格的数据填入文档模板" / "数据驱动生成文档"
- "批量生成合同/通知/报告" / "模板引擎"
- "把这个表的每一行都生成一份文档" / "mail merge"
- "用 Sheet 的数据填充文档" / "Doc template engine"

打通飞书「数据层」（Base/Sheets）和「文档层」（Docs），实现数据驱动的文档自动化生成。典型场景：客户报告、项目通知、合同草稿等需要批量生成的文档。

## 前置条件

支持 **user 身份**。执行前确保已授权：

```bash
lark-cli auth login --domain doc,base,drive
```

## 工作流

```
用户指定模板 ──► docs +fetch（读取模板内容）
                       │
                       ▼
              识别变量占位符 {{xxx}}
                       │
                       ▼
用户指定数据源 ──► base +record-list / sheets
                       │
                       ▼
               字段 ↔ 变量 映射
                       │
                       ▼
              ┌────────┼────────┐
              ▼        ▼        ▼
          单条生成  批量生成  预览模式
                       │
                       ▼
          docs +create（生成文档） / 预览输出
```

### Step 1: 读取模板文档

参考 [`lark-doc/SKILL.md`](https://github.com/larksuite/cli/blob/main/skills/lark-doc/SKILL.md)。

```bash
# 读取模板文档内容
lark-cli docs +fetch --doc "<模板doc_token>" --format markdown
```

从模板内容中识别变量占位符。支持的占位符格式：
- `{{变量名}}` — 标准格式（推荐）
- `{变量名}` — 简化格式
- `【变量名】` — 中文格式

**示例模板：**
```markdown
# {{项目名称}} 项目周报

## 基本信息
- 项目经理：{{负责人}}
- 报告周期：{{开始日期}} ~ {{结束日期}}
- 项目状态：{{状态}}

## 本周进展
{{本周完成内容}}

## 下周计划
{{下周计划内容}}
```

### Step 2: 获取数据源

**方式 A — 多维表格（Base）：**

参考 [`lark-base/SKILL.md`](https://github.com/larksuite/cli/blob/main/skills/lark-base/SKILL.md)。

```bash
# 获取表字段结构
lark-cli base +field-list --base-token "<base_token>" --table-id "<table_id>" --format json

# 获取记录数据
lark-cli base +record-list --base-token "<base_token>" --table-id "<table_id>" \
  --page-size 100 --format json
```

**方式 B — 电子表格（Sheets）：**

参考 [`lark-sheets/SKILL.md`](https://github.com/larksuite/cli/blob/main/skills/lark-sheets/SKILL.md)。

```bash
# 读取电子表格数据（第一行为表头）
lark-cli sheets spreadsheets values get --params '{"spreadsheetToken":"<token>","range":"Sheet1!A1:Z100"}' --format json
```

**方式 C — 用户直接提供数据（单条）：**

用户口述或以 JSON 形式提供变量值，直接进入 Step 4。

### Step 3: 变量映射

展示模板变量与数据源字段的映射关系，供用户确认：

```markdown
## 变量映射确认

| 模板变量 | 数据源字段 | 示例值 |
|---------|-----------|--------|
| {{项目名称}} | 字段"项目名" | XX 系统开发 |
| {{负责人}} | 字段"PM" | 张三 |
| {{状态}} | 字段"当前状态" | 进行中 |
| {{本周完成内容}} | ⚠️ 未匹配 | — |

⚠️ 有 1 个变量未自动匹配。请指定：
- {{本周完成内容}} 对应哪个字段？（或手动输入固定值）
```

**自动映射规则：**
1. 精确匹配：变量名 == 字段名
2. 模糊匹配：变量名包含字段名或字段名包含变量名
3. 未匹配：标注 ⚠️，等用户指定

### Step 4: 生成文档

> **⚠️ 安全规则：批量生成前必须让用户确认映射和生成数量。**

**单条生成：**
```bash
# 替换变量后创建文档
lark-cli docs +create --title "<替换后的标题>" --markdown "<替换后的内容>"
```

**批量生成：**
```bash
# 逐条创建（串行，每条间隔 1 秒）
# 对数据源中的每一行：
#   1. 用该行数据替换模板中的所有变量
#   2. 创建文档
lark-cli docs +create --title "<按行替换的标题>" --markdown "<按行替换的内容>"
```

> **批量安全**：
> - 单次批量最多 **20 条**（避免创建过多文档）
> - 超过 20 条提示用户分批执行
> - 提供预览模式（仅展示替换结果，不创建文档）

**预览模式：**
```markdown
## 预览：第 1 条文档

# XX 系统开发 项目周报

## 基本信息
- 项目经理：张三
- 报告周期：2026-03-25 ~ 2026-03-31
...

---
确认生成？（共 {N} 条，输入"确认"执行）
```

### Step 5: 输出执行报告

```markdown
## 文档生成报告

### 模板信息
- **模板文档**：[{标题}]({链接})
- **数据源**：{Base/Sheets 名称}（共 {N} 条记录）
- **变量**：{M} 个

### 生成结果
| # | 文档标题 | 状态 | 链接 |
|---|---------|------|------|
| 1 | XX 系统开发 项目周报 | ✅ 已创建 | [查看]({链接}) |
| 2 | YY 平台优化 项目周报 | ✅ 已创建 | [查看]({链接}) |
| 3 | ZZ 数据迁移 项目周报 | ✅ 已创建 | [查看]({链接}) |

共生成 **{N}** 篇文档。
```

## 容错机制

| 异常场景 | 处理方式 |
|---------|---------|
| 模板中无占位符 | 提示"未检测到变量占位符"，引导用户在模板中添加 |
| 变量未匹配到字段 | 标注 ⚠️，等用户手动指定或使用固定值 |
| 数据源某行缺失字段 | 将缺失字段替换为"—"，不阻塞 |
| 批量超过 20 条 | 提示分批执行 |
| 文档创建失败 | 记录失败项，继续创建其余文档 |
| 模板文档无权限 | 提示用户检查文档分享设置 |

## 权限表

| 步骤 | 命令 | 所需 scope | 是否必选 |
|------|------|-----------|---------|
| 读取模板 | `docs +fetch` | `docx:document:readonly` | ✅ 必选 |
| Base 数据 | `base +field-list` / `+record-list` | `bitable:bitable:readonly` | 可选（Base 数据源时） |
| Sheets 数据 | `sheets values get` | `sheets:spreadsheet:readonly` | 可选（Sheets 数据源时） |
| 生成文档 | `docs +create` | `docx:document:write` | ✅ 必选 |

## 参考

- [lark-shared](https://github.com/larksuite/cli/blob/main/skills/lark-shared/SKILL.md) — 认证、权限（必读）
- [lark-doc](https://github.com/larksuite/cli/blob/main/skills/lark-doc/SKILL.md) — `+fetch`、`+create` 详细用法
- [lark-base](https://github.com/larksuite/cli/blob/main/skills/lark-base/SKILL.md) — `+field-list`、`+record-list` 详细用法
- [lark-sheets](https://github.com/larksuite/cli/blob/main/skills/lark-sheets/SKILL.md) — 电子表格读取详细用法
