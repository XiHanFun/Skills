---
name: requirement-driven-iterative-change
description: >-
  Decomposes user requests into numbered requirement items after reading the
  codebase, implements changes one item at a time with traceability, rollback
  points, and per-item verification. Use when the user wants structured delivery,
  phased implementation, reversible edits, explicit requirement checklists, or
  when the user writes in Chinese: 需求项、分步改、可追溯、可回滚、可验证.
---

# 需求驱动、分步改代码

先读代码，再拆需求，再逐项实现与验证。不要跳过需求表，也不要在同一步里混入多个无关改动。

## 1. 理解现有代码

在动手改代码前：

1. 定位相关文件，优先看入口、调用方、实现处、测试。
2. 归纳当前行为与约束，写成 3 到 8 条基于代码事实的要点。
3. 对不确定处继续阅读；只有在无法从代码确认时才问一个具体问题。

## 2. 拆解需求

输出带稳定编号的列表，并在本会话内固定使用：

| 字段 | 规则 |
|------|------|
| ID | `REQ-001`、`REQ-002`…（递增） |
| 标题 | 简短动宾短语 |
| 范围 | 可能涉及的文件/模块（尽力而为） |
| 完成标准 | 可观察的验收条件（可测或可感知） |
| 依赖 | 可选：依赖的 `REQ-00x` |

将单项控制到适合一次提交或一批逻辑相关编辑的粒度。模糊需求拆成多条 REQ。

使用这个模板：

```markdown
## 需求项

| ID | 标题 | 范围 | 完成标准 | 依赖 |
|----|------|------|----------|------|
| REQ-001 | … | … | … | — |
```

## 3. 按项实现

按顺序处理每个 `REQ-00n`（遵守依赖关系）：

1. 明确写出“正在实现 `REQ-00n: 标题`”。
2. 只做该 REQ 需要的改动，避免范围蔓延。
3. 保持可追溯：
   - 任务列表中一项 REQ 对应一条 todo。
   - 若使用 git，提交主题包含 `REQ-00n`。
   - 仅在必要时添加简短代码注释说明该行为与 REQ 的关系。

## 4. 保留回滚点

优先保证改动可逆，不依赖用户记忆：

1. 若仓库使用 git，优先每个 REQ 一次提交，或至少保持原子变更批次。
2. 对较大改动优先使用功能分支，全部 REQ 验证通过后再合并。
3. 若无 git，在回复中列出每个 REQ 影响的文件，便于恢复。

## 5. 每项完成后验证

每完成一条需求项，在继续下一条前运行与项目匹配的检查：

- 测试：仓库文档中的单测或集成命令，如 `dotnet test`、`pnpm test`。
- Lint / 类型检查：如 `eslint`、`tsc --noEmit`、解决方案构建。
- 手工验证：仅当自动化无法覆盖时，并写明具体场景。

若验证失败，在同一 REQ 内修复，或拆出新的 `REQ-00m` 跟进。不要留下未声明的坏状态。

## 6. 收尾

全部 REQ 完成后：

1. 给出汇总表：REQ ID → 改动文件或区域 → 验证方式。
2. 给出回滚提示：分支名、提交范围或 `git revert` 建议。

## 反模式

- 不列需求表直接写代码。
- 多条 REQ 混在同一坨 diff 里无法区分。
- REQ 标完成却没有任何验证步骤。
- 大块重构不绑定具体 REQ ID。
