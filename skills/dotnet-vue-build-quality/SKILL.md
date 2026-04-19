---
name: dotnet-vue-build-quality
description: >-
  Runs and interprets builds, tests, lint, and type-check for .NET solutions
  alongside pnpm/npm Vue workspaces. Use when setting up CI, fixing broken
  builds, adding quality gates, or releasing a full-stack .NET + Vue project.
---

# 构建与质量门禁（.NET + Vue）

围绕“能稳定构建、能稳定验证、能稳定发布”组织检查与执行顺序。正文只保留执行建议和排查顺序，不重复触发条件。

## .NET

- 还原与编译：`dotnet restore` → `dotnet build`；发布构建使用 `-c Release`。
- 测试：运行 `dotnet test`，重点看失败测试、并行度、测试数据库和环境变量。
- 格式化与分析：按仓库约定运行 `dotnet format` 或分析器检查；若 `TreatWarningsAsErrors` 生效，按零警告处理。
- 发布产物：输出路径、RID、单文件、裁剪等以仓库文档或项目文件为准。

## 前端（Vue / monorepo）

- 安装：优先使用仓库指定的包管理器，通常是 `pnpm`；遵守 `packageManager` 和 lockfile。
- 常见脚本：`dev`、`build`、`preview`、`lint`、`lint:fix`、`format`、`type-check`、`test`。
- CI 顺序：通常按 `install` → `lint` → `type-check` → `unit test` → `build` 执行，再按团队需要调整。
- Turbo、Nx 等任务编排工具要保证缓存策略与 `.env*` 约定在本地和 CI 一致。

## 全栈流水线

- 并行执行后端 `dotnet test` 与前端 `lint + type-check`，端到端测试放在前后端产物就绪之后。
- 将前端 `dist` 与后端发布包分开展示和归档，静态资源托管与 API 基地址通过配置注入。
- 版本策略优先跟随仓库现状；如有破坏性变更，同步更新 changelog 或迁移说明。

## 故障排查顺序

1. 检查缓存和 lockfile 是否冲突或被错误提交。
2. 检查 Node 和 .NET SDK 版本是否与 `engines`、`global.json` 一致。
3. 检查环境变量和 `.env*` 是否导致本地与 CI 行为不同。
4. 检查路径大小写、LF/CRLF 等跨平台差异。

## 发布前自检

- [ ] 生产构建无错误与未处理警告（按团队策略）  
- [ ] 关键路径有自动化测试或手动清单  
- [ ] 配置与密钥未进入仓库  
