---
name: dotnet-vue-fullstack-dev
description: >-
  Guides layered ASP.NET Core APIs with Vue 3 + TypeScript SPAs: contracts,
  validation, auth boundaries, and UI data flow. Use when designing features,
  adding endpoints, wiring Pinia/composables, or splitting responsibilities
  between backend and frontend in a .NET + Vue codebase.
---

# 全栈开发：.NET API + Vue 前端

## 后端（API）

- **分层**：Web 层只负责 HTTP、认证授权、模型绑定与结果映射；业务规则与持久化放在应用/领域与基础设施边界内，避免控制器膨胀。
- **契约**：请求/响应 DTO 与领域实体分离；公开 API 使用稳定、版本化的形状，内部实现细节不泄漏到 JSON。
- **验证**：入参在边界集中校验（数据注解、FluentValidation 等择一或组合），失败返回 **可预期的错误结构与 HTTP 状态码**。
- **异步**：I/O 使用 `async`/`await` 到底；避免在请求路径上 `Task.Result` / `.Wait()`。
- **安全**：默认鉴权；最小权限；敏感配置来自环境/密钥管理，不进仓库；日志中不落密码与令牌。
- **错误**：区分客户端错误（4xx）与服务端故障（5xx）；必要时带 **业务错误码** 便于前端统一处理。

## 前端（Vue 3 + TypeScript）

- **状态**：路由级与跨页共享用 **Pinia**（或项目选定方案）；组件内局部状态用 `ref`/`reactive`，避免全局滥用。
- **数据获取**：API 调用集中在 **api 模块或服务层**，组件只编排 UI 与调用结果；统一处理 loading / 错误 / 空状态。
- **类型**：优先 TypeScript；与后端契约对齐的 **接口或生成类型**（OpenAPI 等）减少漂移。
- **组件**：可复用 UI 与页面容器分离；列表/表单模式保持 props 与事件清晰，避免隐式跨组件可变全局。
- **可访问性与 UX**：关键操作有禁用与反馈；危险操作需确认；与后端错误消息映射为用户可读文案。

## 功能串联 checklist

- [ ] API 契约、状态码与错误体与前端处理一致  
- [ ] 权限：后端强制校验，前端仅做 UX 隐藏  
- [ ] 分页/排序/筛选参数命名前后端一致  
- [ ] 并发与重复提交（按钮节流、幂等键）按需处理  

## 文档与配置站点

- 框架级、组件库、业务应用可能分仓库：开发时以 **当前仓库 README 与脚本** 为准，不假设全局路径。
