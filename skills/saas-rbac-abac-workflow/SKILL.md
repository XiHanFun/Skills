---
name: saas-rbac-abac-workflow
description: >-
  B2B SaaS 多租户 RBAC + ABAC 全流程规范（ASP.NET Core + Vue 3）。
  涵盖架构/DBA/后端/前端/安全/测试/DevOps/PM 八角色交付物。
  触发词：多租户, 租户隔离, 权限矩阵, RBAC, ABAC, 字段脱敏, 越权, 灰度发布, 数据范围, 权限码.
---

# SaaS 多租户 RBAC + 轻量 ABAC 全流程规范

本技能面向 B2B SaaS 多租户系统，提供从需求到上线的各角色（架构师、DBA、后端、前端、安全、测试、DevOps、PM）规范化交付物模板与检查清单。技术栈基于 ASP.NET Core + Vue 3 + TypeScript，同时保留通用原则。

## 使用时机

出现以下任一信号时，按本技能执行：

- 新建或改造多租户功能（数据、菜单、API）
- 设计或修改 RBAC 角色、权限、ABAC 策略
- 审查字段级安全、脱敏、数据范围
- 规划权限矩阵测试、租户隔离回归
- 规划灰度发布、租户级配置、回滚
- 审核或拆分与权限相关的需求

## 核心原则（所有角色必须遵守）

1. **租户隔离第一**：任何查询、写入、缓存、消息、文件存储都必须显式携带 `TenantId`；禁止"默认共享"。
2. **最小权限**：默认拒绝；先有 `Permission`，再挂 `Role`，最后挂 `User`；ABAC 仅用于补充 RBAC 无法表达的维度（数据范围、字段可见性、时间/IP/设备）。
3. **权限表达统一**：权限码采用 `模块:资源:动作` 三段式（示例 `saas:user:read`）；禁止硬编码散落前后端。
4. **数据范围显式化**：列表/详情一律走"数据范围过滤器"，不得在 UI 层二次裁剪敏感数据。
5. **审计全覆盖**：权限授予/撤销、敏感字段访问、跨租户操作必须审计，且审计日志本身受 RBAC 保护。
6. **可灰度、可回滚**：权限模型变更、策略调整必须支持按租户灰度，并保留 1 个版本的回滚脚本。

## 权限码命名约定

| 段位 | 含义 | 示例 |
|------|------|------|
| 1 | 业务模块 | `saas`, `crm`, `billing` |
| 2 | 资源 | `user`, `role`, `order` |
| 3 | 动作 | `read`, `create`, `update`, `delete`, `export`, `approve` |

派生操作追加第 4 段（可选）：`saas:user:export:sensitive`。

## 数据范围（Data Scope）标准值

| 范围 | 说明 |
|------|------|
| `Self` | 仅本人创建 |
| `Dept` | 本部门 |
| `DeptAndChild` | 本部门及下级 |
| `Tenant` | 本租户（默认最大范围） |
| `Custom` | 自定义组织/部门 ID 集合 |
| `Global` | 仅平台超管，跨租户 |

## 角色 × 阶段交付物索引

查 **"我是什么角色 + 在哪个阶段"** 即可定位详细规范文件：

| 阶段 \ 角色 | 架构师 | DBA | 后端 | 前端 | 安全 | 测试 | DevOps | PM |
|-------------|:------:|:---:|:----:|:----:|:----:|:----:|:------:|:--:|
| 需求/设计 | A | D | B | F | S | — | — | P |
| 开发 | A | D | B | F | S | — | — | — |
| Code Review | A | D | B | F | S | — | — | — |
| 测试 | — | D | B | F | S | T | — | P |
| 发布 | A | D | B | F | S | T | O | P |
| 运维/运营 | A | D | B | — | S | — | O | P |

字母对应下列分册（均为一级引用）：

- A 架构师交付物 → [architecture.md](architecture.md)
- D 数据库设计交付物 → [database.md](database.md)
- B 后端开发交付物 → [backend.md](backend.md)
- F 前端开发交付物 → [frontend.md](frontend.md)
- S 安全工程交付物 → [security.md](security.md)
- T 测试工程交付物 → [testing.md](testing.md)
- O DevOps 交付物 → [devops.md](devops.md)
- P 产品/项目经理交付物 → [product.md](product.md)
- 各阶段合并门禁清单 → [checklists.md](checklists.md)

## 主工作流

对每一个涉及权限/多租户的任务，按下列工作流执行：

```
阶段门禁：
- [ ] 1. 需求拆分：PM 产出"权限矩阵草案 + 数据范围假设"（product.md）
- [ ] 2. 架构评审：架构师确认隔离方案与扩展点（architecture.md）
- [ ] 3. 数据建模：DBA 给出 TenantId 策略、索引、迁移脚本（database.md）
- [ ] 4. 后端实现：按模板实现 AuthZ 中间件/策略处理器（backend.md）
- [ ] 5. 前端实现：按指令/路由守卫落地菜单/按钮/数据权限（frontend.md）
- [ ] 6. 安全评审：字段脱敏、审计、越权用例覆盖（security.md）
- [ ] 7. 测试：权限矩阵用例 + 租户隔离回归全绿（testing.md）
- [ ] 8. 发布：灰度 + 回滚预案就绪（devops.md）
- [ ] 9. 运营：监控、审计看板、工单入口就位（devops.md + security.md）
```

任一门禁未通过，禁止进入下一阶段。

## 多租户隔离模式（供架构师选择）

| 模式 | 适用 | 代价 |
|------|------|------|
| 共享库共享表（TenantId 列） | 中小 B2B、租户数多、数据量均匀 | 风险：漏加过滤器即越权 |
| 共享库分表（按租户分表） | 单租户数据量大、查询热点明显 | 迁移/索引复杂 |
| 分库（每租户独立库） | 合规/隔离要求高、KA 客户 | 运维成本高、跨租户统计困难 |
| 混合（平台库 + 租户库） | 大 B2B，平台元数据与业务数据分离 | 连接切换/事务边界复杂 |

**默认推荐**：共享库共享表 + 全局查询过滤器 + `TenantId` 强制索引；对 KA 客户支持按租户切换到独立库。

## ABAC 轻量策略表达

采用"声明式属性 + 条件表达式"模式，**不引入重型策略引擎**：

```json
{
  "policyId": "order.read.dept-scope",
  "effect": "allow",
  "subject": { "hasPermission": "crm:order:read" },
  "resource": "order",
  "condition": {
    "all": [
      { "eq": ["resource.tenantId", "subject.tenantId"] },
      { "in": ["resource.deptId", "subject.dataScope.deptIds"] },
      { "notIn": ["resource.status", ["Archived"]] }
    ]
  }
}
```

条件算子仅支持：`eq`, `ne`, `in`, `notIn`, `gt`, `gte`, `lt`, `lte`, `all`, `any`, `not`。新增算子需架构师评审。

## 字段级安全（FLS）

基于本仓库 `SysFieldLevelSecurity` + `FieldMaskStrategy` + `FieldSecurityTargetType`：

- 目标：角色 / 用户 / 部门任一维度
- 策略：`None / FullMask / PartialMask / Hash / Encrypted / Deny`
- 实施点：**仅在序列化出口与导出出口**执行；不得依赖前端过滤

详见 [security.md](security.md)。

## 何时进入某个分册

- 只是"加一个菜单 + 一个列表接口"：读 backend.md + frontend.md + checklists.md
- 新增一个业务模块：按索引读 architecture.md → database.md → backend.md → frontend.md → testing.md
- 做合规/安全审计：读 security.md + checklists.md
- 准备上线或回滚：读 devops.md + checklists.md
- 梳理需求：读 product.md

## 通用反模式（红线）

- ❌ 在 Controller 里手写 `if (user.TenantId != entity.TenantId)` 校验
- ❌ 在前端用 `v-if="user.isAdmin"` 判定业务权限（仅限 UI 展示）
- ❌ 把权限码拼在前端常量里而不走后端下发
- ❌ 用 `SELECT *` + 前端脱敏代替 FLS
- ❌ 把 `TenantId` 放进 URL 路径或 Query 参与鉴权（必须来自可信上下文：JWT/Session）
- ❌ 迁移脚本中无 `WHERE TenantId = ?` 的批量更新
- ❌ 发布不做灰度、回滚脚本缺失或未演练

## 快速自检（任何变更合入前）

```
- [ ] 所有新表含 TenantId 且建立 (TenantId, ...) 复合索引
- [ ] 所有查询走全局过滤器或显式租户上下文
- [ ] 新增权限码已在 PermissionSeed 登记并下发
- [ ] 菜单/按钮/数据范围三层权限齐备
- [ ] 敏感字段走 FLS，非前端裁剪
- [ ] 审计日志覆盖授权变更 + 敏感读写
- [ ] 至少包含：正权限 / 负权限 / 跨租户越权 三类测试
- [ ] 灰度开关与回滚脚本已就绪
```

## 进一步阅读

- [architecture.md](architecture.md) — 架构师：隔离模式、权限模型、扩展点
- [database.md](database.md) — DBA：TenantId 策略、索引、迁移与分库分表
- [backend.md](backend.md) — 后端：AuthN/AuthZ 中间件、策略处理器、代码模板
- [frontend.md](frontend.md) — 前端：路由守卫、指令、数据范围联动
- [security.md](security.md) — 安全：FLS、审计、越权清单、脱敏策略
- [testing.md](testing.md) — 测试：权限矩阵、隔离回归、用例模板
- [devops.md](devops.md) — DevOps：租户配置、灰度、回滚、监控
- [product.md](product.md) — PM：需求模板、权限矩阵草案、验收标准
- [checklists.md](checklists.md) — 各阶段合并门禁清单

## 用户需求

以下是用户提供的需求或上下文，请据此判断应进入哪个分册并执行对应规范：

$ARGUMENTS
