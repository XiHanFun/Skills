---
name: dotnet-vue-code-style
description: >-
  .NET + Vue 全栈代码风格规范：EditorConfig、Prettier、缩进、命名约定与冲突处理。
  触发词：代码风格, 格式化, EditorConfig, Prettier, 缩进, 命名规范, lint 冲突.
---

# .NET 与 Vue 代码格式（全局约定）

## 原则

- 仓库根目录用 `.editorconfig` 统一换行与编码；**后端 C# 与前端 Web 资源缩进不同是常态**，不要强行混用同一套 indent 给两种语言。
- 提交前运行各侧的 **format** 脚本（或 `dotnet format` / Prettier），避免在 PR 里夹杂无关格式抖动。

## C# / .NET

- **编码**：UTF-8；**换行**：LF；**文件末行**：保留单个换行。
- **缩进**：4 空格；**命名空间**：优先 file-scoped（若团队关闭主构造函数等规则，以仓库 `.editorconfig` / `Directory.Build.props` 为准）。
- **命名**：接口 `I` 前缀；类型与公共成员 PascalCase；私有实例字段 `_camelCase`；静态只读等以团队规则为准。
- **风格倾向**：`var` 在类型明显处；优先 `using` 简化形式；需要时显式括号；集合表达式、模式匹配、null 条件按团队 severity 执行。
- **与工具链**：以 **IDE 分析器 + `.editorconfig`** 为单一事实来源；改风格时同步改配置，不要只改局部文件。

## Vue / TypeScript / 前端资源

- **缩进**：2 空格（与 C# 的 4 空格并存）。
- **Prettier 常见默认**：无分号、`singleQuote: true`、`printWidth` 约 100、`trailingComma: 'all'`、`endOfLine: 'lf'`、`vueIndentScriptAndStyle: false`（具体以项目配置为准）。
- **Markdown**：常对 `trim_trailing_whitespace` 单独放宽，避免破坏有意尾随空格。
- **包管理**： monorepo 常见 **pnpm** + `packageManager` 字段；遵守 `engines` 与 lockfile 策略。

## 冲突处理

- 同一文件既有 ESLint 又有 Prettier：优先 **eslint-config-prettier** 关闭冲突规则，由 Prettier 管格式。
- C# 与脚本生成代码：生成物纳入忽略或单独目录，避免全量 format 破坏生成器输出。

## 何时查阅

- 新增子项目、合并多仓库风格、或 CI 报 format 失败时，先对齐上述层级再改业务代码。

## 相关 skill

- `/dotnet-vue-fullstack-dev` — 全栈开发指南
- `/dotnet-vue-build-quality` — 构建与质量门禁（含 format 脚本）

## 用户需求

以下是用户提供的代码或风格问题，请按上述规范给出建议：

$ARGUMENTS
