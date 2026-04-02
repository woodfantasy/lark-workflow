# Skill 设计规范指南

本文档总结 lark-workflow 项目的 Skill 编写范式，供贡献者参考。

## 1. SKILL.md 结构模板

```markdown
---
name: lark-workflow-<name>
version: 1.0.0
description: "<功能描述>。当用户需要<触发场景>时使用。"
metadata:
  requires:
    bins: ["lark-cli"]
---

# <标题>工作流

**CRITICAL — 开始前 MUST 先用 Read 工具读取 lark-shared/SKILL.md**

## 适用场景
- <自然语言触发词列表>

## 前置条件
- 身份要求（user / bot）
- 授权命令

## 工作流
- ASCII art 数据流图
- 分步骤说明（Step 1, 2, 3...）

## 输出模板
- 结构化 Markdown 格式

## 权限表
- scope 对照表

## 参考
- 引用的原子 Skill 链接
```

## 2. 核心设计原则

### 2.1 时间处理
- **禁止** AI 心算日期，必须使用 `date` 命令
- 时间参数必须使用 ISO 8601 格式
- 提供 macOS / Linux 两种 `date` 命令写法

### 2.2 数据降级
- 每个非核心域都有"降级处理"说明
- 某个域执行失败不影响整体流程
- 在输出中标注未授权的模块

### 2.3 安全确认
- 写入操作（创建任务、发送消息、创建文档）前确认用户意图
- 建议使用 `--dry-run` 预览
- **绝对禁止**在未经用户确认时执行批量写入

### 2.4 上下文管理
- 大数据量场景提供截断策略
- 使用 `--format json` 便于 AI 解析
- 使用 `--limit` / `--page-size` 控制返回数据量

### 2.5 跨域引用
- 原子 Skill 通过 GitHub 绝对链接引用
- 不假设用户本地安装了哪些 Skill
- 在"参考"章节统一列出所有引用

## 3. 命名规范

- Skill 名称格式：`lark-workflow-<feature>`
- 使用小写字母和连字符
- 名称应直观反映功能（daily-briefing, weekly-report, action-extractor）

## 4. 版本管理

- 遵循语义化版本（SemVer）
- 新增功能：minor 版本 +1
- 破坏性变更：major 版本 +1
- Bug 修复：patch 版本 +1
