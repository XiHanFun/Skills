---
name: dotnet-vue-build-quality
description: >-
  Runs and interprets builds, tests, lint, and type-check for .NET solutions
  alongside pnpm/npm Vue workspaces. Use when setting up CI, fixing broken
  builds, adding quality gates, or releasing a full-stack .NET + Vue project.
---

# 构建与质量门禁（.NET + Vue）

## .NET

- **还原与编译**：`dotnet restore` → `dotnet build`（Release 用 `-c Release`）。
- **测试**：`dotnet test`；关注失败测试、并行度、测试数据库/环境变量。
- **格式化/分析**：`dotnet format`（若仓库启用）与编译器/分析器警告；TreatWarningsAsErrors 时零警告策略。
- **产物**：发布输出路径、运行时标识符（RID）、单文件/裁剪等以项目文档为准。

## 前端（Vue / monorepo）

- **安装**：优先 **pnpm**（或仓库规定的包管理器），遵守 `packageManager` 与 lockfile。
- **脚本常见名**：`dev`、`build`、`preview`、`lint` / `lint:fix`、`format`、`type-check` / `vue-tsc --noEmit`、`test`。
- **CI 顺序建议**：`install` → `lint` → `type-check` → `unit test` → `build`（与团队调整）。
- **Turbo/Nx 等**：使用过滤任务与缓存时，确保 **环境变量与 `.env*` 策略** 在 CI 中一致。

## 全栈流水线（思路）

- **并行**：后端 `dotnet test` 与前端 `lint + type-check` 可并行；**端到端** 依赖前后端 artifact 或容器编排。
- **制品**：前端 `dist` 与后端发布包分开展示；静态资源托管与 API 基地址由配置注入。
- **版本**：语义化版本；changelog/迁移说明与破坏性变更同步。

## 故障排查顺序

1. 清缓存与 lockfile 是否被错误提交或冲突  
2. Node/.NET SDK 版本与 `engines` / `global.json` 是否匹配  
3. 环境变量缺失导致构建时行为不同  
4. 路径大小写 / LF 在跨平台 CI 上的问题  

## 发布前自检

- [ ] 生产构建无错误与未处理警告（按团队策略）  
- [ ] 关键路径有自动化测试或手动清单  
- [ ] 配置与密钥未进入仓库  
