---
name: lark-workflow-base-analytics
version: 1.0.0
description: >
  多维表格智能分析：读取飞书多维表格的结构和数据，进行趋势分析、异常检测和数据洞察，
  生成结构化分析报告。当用户需要分析多维表格数据、查看数据趋势、
  或问「帮我分析一下这个表」时使用。
metadata:
  requires:
    bins:
      - lark-cli
---

# 多维表格智能分析工作流

**CRITICAL — 开始前 MUST 先用 Read 工具读取 [`lark-shared/SKILL.md`](https://github.com/larksuite/cli/blob/main/skills/lark-shared/SKILL.md)，其中包含认证、权限处理和安全规则。**

## 适用场景

- "帮我分析一下这个表" / "这个多维表格有什么趋势"
- "给这个 Base 做个数据报告" / "分析一下这个表的数据"
- "这个表里有没有异常数据" / "多维表格数据洞察"
- "帮我看看这个表的分布情况" / "数据仪表盘"
- "汇总一下这个表的关键指标" / "Base analytics"

## 核心价值

多维表格（Base/Bitable）是飞书相对竞品最核心的差异化产品。本 Skill 让 AI 能读懂表结构、聚合分析数据、检测异常并生成洞察报告——释放沉淀在 Base 中的数据价值。

## 前置条件

支持 **user 身份**。执行前确保已授权：

```bash
lark-cli auth login --domain base,doc
```

## 工作流

```
用户指定 Base ──► 解析 app_token / table_id
                       │
                       ▼
              base +field-list（获取表结构）
                       │
                       ▼
              base +data-query（聚合分析）
              base +record-list（取样本数据）
                       │
                       ▼
              AI 分析引擎
              ┌────────┼────────┐
              ▼        ▼        ▼
           趋势分析  异常检测  分布统计
                       │
                       ▼
              结构化分析报告（可选：docs +create）
```

### Step 1: 解析数据源

用户可能提供的输入：

**A. Base 链接：**
```
https://xxx.feishu.cn/base/<app_token>?table=<table_id>&view=<view_id>
```
提取 `app_token`、`table_id`、`view_id`。

**B. 直接提供 app_token + table_id：**

直接进入 Step 2。

**C. 模糊描述（"那个销售数据表"）：**

提示用户提供具体链接或 token。

### Step 2: 获取表结构

参考 [`lark-base/SKILL.md`](https://github.com/larksuite/cli/blob/main/skills/lark-base/SKILL.md)。

```bash
# 列出所有数据表
lark-cli base +table-list --app-token "<app_token>" --format json

# 获取目标表的字段结构
lark-cli base +field-list --app-token "<app_token>" --table-id "<table_id>" --format json
```

从字段列表中识别：
- **数值字段**（Number / Currency / Progress）→ 可聚合分析
- **日期字段**（DateTime / CreatedTime / ModifiedTime）→ 可做时间趋势
- **选项字段**（SingleSelect / MultiSelect）→ 可做分布统计
- **人员字段**（User）→ 可做责任人维度分析
- **公式/查找字段**（Formula / Lookup）→ 注意只读

### Step 3: 数据聚合分析

```bash
# 使用 data-query 进行聚合（推荐方式）
lark-cli base +data-query --app-token "<app_token>" --table-id "<table_id>" \
  --field-names '["<字段1>","<字段2>"]' \
  --group-by '["<分组字段>"]' \
  --aggregations '[{"field_name":"<数值字段>","type":"sum"}]' \
  --format json
```

**常用聚合类型：**
- `sum` — 求和（金额、数量）
- `average` — 平均值
- `max` / `min` — 最大/最小值
- `count` — 计数
- `counta` — 非空计数

> **注意**：`+data-query` 支持 `--filter` 参数进行条件过滤。使用 `--page-size` 控制返回量。

```bash
# 获取记录样本（当需要逐行分析时）
lark-cli base +record-list --app-token "<app_token>" --table-id "<table_id>" \
  --page-size 100 --format json
```

> **⚠️ 数据量控制**：Base 单次最多返回 500 条记录。如果数据量大，使用 `--page-token` 分页获取，但分析场景通常用聚合查询（`+data-query`）更高效。

### Step 4: AI 分析引擎

基于表结构和聚合结果，AI 执行以下分析：

**4.1 趋势分析（有日期字段时）：**
- 按日/周/月聚合数值字段
- 识别增长/下降趋势
- 计算环比/同比变化率

**4.2 分布分析（有选项字段时）：**
- 各选项值的数量和占比
- 识别集中度（是否严重偏斜）

**4.3 异常检测：**
- 数值字段中的极端值（>3σ）
- 缺失值比例
- 数据一致性检查（如日期字段是否有未来日期）

**4.4 关联分析（多个数值字段时）：**
- 字段间的相关性推断
- 交叉分组对比

### Step 5: 生成分析报告

```markdown
## 📊 多维表格分析报告

### 数据概览
- **表名**：{表名}
- **字段数**：{N} 个（{M} 个数值字段，{P} 个选项字段）
- **记录总数**：{count} 条
- **时间跨度**：{最早日期} ~ {最晚日期}

### 📈 关键指标
| 指标 | 值 | 趋势 |
|------|---|------|
| {数值字段1} 总计 | ¥1,234,567 | 📈 +12.3% 环比 |
| {数值字段2} 平均 | 85.6 | 📉 -3.2% 环比 |
| 记录增长 | 本月 +45 条 | 📈 |

### 📊 分布分析
**{选项字段名} 分布：**
| 值 | 数量 | 占比 | 可视化 |
|----|------|------|--------|
| 进行中 | 45 | 38% | ████████░░ |
| 已完成 | 52 | 44% | █████████░ |
| 已取消 | 21 | 18% | ████░░░░░░ |

### 📈 时间趋势
**{数值字段} 月度趋势：**
| 月份 | 值 | 环比变化 |
|------|---|---------|
| 1月 | 120,000 | — |
| 2月 | 135,000 | +12.5% |
| 3月 | 128,000 | -5.2% |

### ⚠️ 异常与问题
- ⚠️ "{字段X}" 有 {N} 条空值（占比 {P}%）
- ⚠️ 记录 #{id} 的 "{字段Y}" 值异常偏高（{值}，是平均值的 {倍数} 倍）
- ⚠️ 最近 7 天无新增记录

### 💡 洞察与建议
1. **{洞察 1}**：基于趋势数据推导的结论
2. **{洞察 2}**：基于分布数据推导的建议
3. **{洞察 3}**：基于异常检测的告警或建议行动
```

### Step 6: 写回文档（可选，用户要求时）

参考 [`lark-doc/SKILL.md`](https://github.com/larksuite/cli/blob/main/skills/lark-doc/SKILL.md)。

```bash
lark-cli docs +create --title "数据分析报告：{表名} (<date>)" --markdown "<报告内容>"
```

## 容错机制

| 异常场景 | 处理方式 |
|---------|---------|
| 表无数值字段 | 仅做分布分析和记录统计，不做趋势分析 |
| 表无日期字段 | 跳过时间趋势分析，标注"无时间维度" |
| 数据量 >500 条 | 使用 `+data-query` 聚合查询，不逐行获取 |
| 表为空（0 条记录） | 提示"该表暂无数据" |
| 字段类型无法聚合 | 跳过该字段，分析其他可聚合字段 |
| app_token 无权限 | 提示用户检查 Base 分享权限 |

## 权限表

| 步骤 | 命令 | 所需 scope | 是否必选 |
|------|------|-----------|---------|
| 表结构 | `base +table-list` / `+field-list` | `bitable:bitable:readonly` | ✅ 必选 |
| 数据聚合 | `base +data-query` | `bitable:bitable:readonly` | ✅ 必选 |
| 记录列表 | `base +record-list` | `bitable:bitable:readonly` | 可选（逐行分析时） |
| 生成报告 | `docs +create` | `docx:document:write` | 可选（写回时） |

## 参考

- [lark-shared](https://github.com/larksuite/cli/blob/main/skills/lark-shared/SKILL.md) — 认证、权限（必读）
- [lark-base](https://github.com/larksuite/cli/blob/main/skills/lark-base/SKILL.md) — `+field-list`、`+data-query`、`+record-list` 详细用法
- [lark-doc](https://github.com/larksuite/cli/blob/main/skills/lark-doc/SKILL.md) — `+create` 详细用法
