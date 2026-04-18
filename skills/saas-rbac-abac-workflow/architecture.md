# 架构师交付物规范

架构师是多租户 + RBAC/ABAC 体系的"宪法制定者"。任何新模块/重大变更必须先拿到架构评审通过。

## 核心职责

1. 选定并声明租户隔离模式（共享表 / 分表 / 分库 / 混合）
2. 定义权限模型与扩展点（RBAC 骨架 + ABAC 切入点）
3. 规划跨模块边界（平台库 vs 租户库、主从、缓存、消息、文件）
4. 维护权限码命名空间、ABAC 条件算子清单
5. 对破坏性变更拍板迁移与灰度策略

## 交付物（ADR 模板）

每个重大决策必须产出一份 ADR（Architecture Decision Record）：

```markdown
# ADR-NNN: [决策标题]

## 背景
- 业务诉求 / 合规要求 / 规模假设

## 决策
- 明确结论：采用 X 模式
- 生效范围与例外

## 备选方案对比
| 方案 | 优点 | 缺点 | 成本 |

## 影响面
- 数据层 / 服务层 / 前端 / DevOps
- 迁移与回滚策略
- 性能与安全影响

## 验收标准
- 可量化的指标：RT / QPS / 隔离违规数 = 0
```

## 租户隔离模式选择矩阵

| 维度 | 共享表 | 分表 | 分库 | 混合 |
|------|:------:|:----:|:----:|:----:|
| 租户数 > 1000 | ✅ | ⚠️ | ❌ | ⚠️ |
| 单租户数据 > 亿行 | ❌ | ✅ | ✅ | ✅ |
| 强合规/物理隔离 | ❌ | ❌ | ✅ | ✅ |
| 跨租户统计 | ✅ | ✅ | ❌ | ⚠️ |
| 运维复杂度 | 低 | 中 | 高 | 高 |

## 权限模型骨架（必须落地的实体）

```
Tenant ── TenantUser ── User
            │
            └── UserRole ── Role ── RolePermission ── Permission
                                        │
                                        └── RoleDataScope（数据范围）
                                        │
                                        └── FieldLevelSecurity（字段安全）

AbacPolicy ── PolicyBinding（可绑定到 Role / User / Tenant 维度）
```

最小表集（逻辑）：
- `SysTenant` / `SysUser` / `SysRole` / `SysPermission`
- `SysUserRole` / `SysRolePermission`
- `SysRoleDataScope`
- `SysFieldLevelSecurity`
- `SysAbacPolicy` / `SysAbacPolicyBinding`
- `SysAuditLog`

## ABAC 切入点规范

**允许扩展的维度**：
- `subject.*`：用户属性（部门、岗位、数据范围、标签）
- `resource.*`：资源属性（租户、部门、状态、创建人、敏感等级）
- `env.*`：环境属性（时间、IP 段、设备指纹、来源渠道）

**禁止**：
- 在 ABAC 条件里调用外部 HTTP / DB（必须走属性提供器 `IAttributeProvider`）
- 在 ABAC 条件里表达"登录成功"类 AuthN 逻辑
- 将业务规则（如"订单金额 > 10000 需审批"）写成 ABAC 策略（走工作流引擎）

## 扩展点（Extension Points）

| 扩展点 | 目的 | 实现形式 |
|--------|------|----------|
| `ITenantResolver` | 从请求解析 `TenantId` | DI 接口 |
| `IPermissionProvider` | 权限码注册 | 启动期扫描 + Seed |
| `IDataScopeProvider` | 数据范围解析 | DI 接口 |
| `IAttributeProvider` | ABAC 属性获取 | DI + 缓存 |
| `IAuthorizationHandler` | 策略处理器 | ASP.NET Core 原生 |
| `IFieldMasker` | 字段脱敏策略 | DI + 策略模式 |
| `IAuditSink` | 审计输出 | DI，多实现聚合 |

## 缓存与一致性

- 权限缓存键：`perm:{tenantId}:{userId}`，TTL ≤ 10 分钟
- 权限变更必须发 `PermissionChangedEvent` 主动失效
- 跨节点失效通过 Redis Pub/Sub 或消息总线
- ABAC 策略缓存键：`abac:{tenantId}:{policyId}`，版本号驱动失效

## 架构评审检查清单

```
- [ ] 隔离模式已 ADR 化，例外明确
- [ ] 新模块权限码命名空间已在平台注册
- [ ] 所有跨模块调用显式携带 TenantId（RPC 头 / 消息头）
- [ ] 新增 ABAC 条件算子经评审，未突破白名单
- [ ] 缓存失效路径闭环
- [ ] 性能基线：授权判定 p99 ≤ 2ms（本地缓存命中）
- [ ] 灾难恢复：权限服务降级策略明确（默认拒绝 vs 兜底放行）
- [ ] 审计链路完整，可追溯到单次请求
```

## 常见架构反模式

- ❌ 把 `TenantId` 放到连接字符串动态切换但没有连接池隔离
- ❌ 平台超管角色与租户角色混在同一张 `Role` 表无区分字段
- ❌ 缓存 key 不含 `TenantId` 导致租户串数据
- ❌ 消息队列 Topic 未按租户分区，消费端靠手工过滤
- ❌ ABAC 与 RBAC 解释顺序不明确（先 RBAC deny 还是先 ABAC allow）

## 解释顺序（必须明确写入文档）

```
1. AuthN：解析身份 → 拿到 subject
2. TenantCheck：subject.TenantId 与 resource.TenantId 一致（平台超管除外）
3. RBAC：检查 subject 是否拥有 permission
4. DataScope：按 subject 数据范围过滤资源集合
5. ABAC：对单资源进行条件判定（可 deny，可补 allow）
6. FLS：在序列化出口进行字段脱敏
7. Audit：敏感操作落审计
```

任一步失败即拒绝；**deny 优先于 allow**。
