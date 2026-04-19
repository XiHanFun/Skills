---
name: saas-rbac-abac-workflow:database
description: >-
  DBA 交付物：TenantId 策略、复合索引规范、全局查询过滤器、迁移脚本模板、
  分库分表路由与审计表归档。用于数据库设计评审与迁移。
---

# 数据库设计师（DBA）交付物规范

DBA 负责把架构师确定的隔离模式落成具体的表结构、索引、迁移与回滚脚本。

## 核心职责

1. 为所有业务表植入 `TenantId` 并建立合理索引
2. 审核查询计划，避免"漏过滤"造成全表扫描或越权
3. 提供向前迁移 + 回滚脚本，支持按租户灰度
4. 分库分表方案落地与路由规则维护
5. 审计 / 权限相关表的容量与归档策略

## 表设计规范

### 必备列（所有租户级业务表）

| 列 | 类型 | 说明 |
|----|------|------|
| `Id` | `bigint` / `uuid` | 主键，建议雪花 ID |
| `TenantId` | `bigint` | **非空**，默认索引首列 |
| `CreatedBy` / `CreatedAt` | `bigint` / `datetime2` | 审计 |
| `UpdatedBy` / `UpdatedAt` | `bigint` / `datetime2` | 审计 |
| `IsDeleted` | `bit` | 软删除 |
| `RowVersion` | `rowversion` / `timestamp` | 乐观锁 |

### 索引规范

- **复合索引第一列必须是 `TenantId`**（除跨租户平台表）
- 唯一约束带 `TenantId`：`UNIQUE (TenantId, Code)`
- 软删除字段进入过滤索引：`WHERE IsDeleted = 0`
- 外键同租户校验：应用层保证，DB 仅索引

示例：

```sql
CREATE UNIQUE INDEX UX_SysUser_TenantId_UserName
    ON SysUser (TenantId, UserName)
    WHERE IsDeleted = 0;

CREATE INDEX IX_Order_TenantId_Status_CreatedAt
    ON [Order] (TenantId, Status, CreatedAt DESC)
    WHERE IsDeleted = 0;
```

### 主键与分布

- 共享表模式：使用**雪花 ID**，避免自增主键在分库时冲突
- 雪花 ID 的 workerId 保留 2~3 位用于"租户分片"可扩展性
- 禁止使用 `(TenantId, SeqNo)` 复合主键（迁移/外键负担重）

## 全局查询过滤器（EF Core 示例）

所有实体继承抽象基类：

```csharp
public abstract class TenantEntity
{
    public long Id { get; set; }
    public long TenantId { get; set; }
    public bool IsDeleted { get; set; }
}
```

DbContext 中统一注册：

```csharp
protected override void OnModelCreating(ModelBuilder mb)
{
    foreach (var et in mb.Model.GetEntityTypes()
        .Where(t => typeof(TenantEntity).IsAssignableFrom(t.ClrType)))
    {
        var param = Expression.Parameter(et.ClrType, "e");
        var tenantProp = Expression.Property(param, nameof(TenantEntity.TenantId));
        var deletedProp = Expression.Property(param, nameof(TenantEntity.IsDeleted));
        var currentTenant = Expression.Property(
            Expression.Constant(_tenantContext), nameof(ITenantContext.TenantId));

        var body = Expression.AndAlso(
            Expression.Equal(tenantProp, currentTenant),
            Expression.Equal(deletedProp, Expression.Constant(false)));
        mb.Entity(et.ClrType).HasQueryFilter(Expression.Lambda(body, param));
    }
}
```

**规则**：
- 平台超管场景通过 `IgnoreQueryFilters()` 显式开闸，禁止"默认忽略"
- 跨租户统计走独立只读库 / CQRS 读模型

## 迁移脚本规范

### 命名

```
YYYYMMDD_HHmm__<module>__<change>.sql
例：20260417_1030__saas__add_role_datascope.sql
```

每次迁移必须包含：
1. `up.sql`：向前变更
2. `down.sql`：回滚脚本（能跑通）
3. `verify.sql`：验证 SQL（行数、约束、索引）

### 批量 DML 强制要求

```sql
-- 必须显式 WHERE TenantId
UPDATE [Order]
SET Status = 'Archived'
WHERE TenantId = @TenantId
  AND CreatedAt < @Cutoff;
```

禁止无 `WHERE TenantId` 的批量更新/删除（CI 中通过正则扫描拦截）。

### 大表变更

- 在线 DDL：`ALGORITHM=INPLACE, LOCK=NONE`（MySQL）/ `ONLINE=ON`（SQL Server）
- 超大表：使用 `pt-online-schema-change` / `gh-ost` / Azure Online
- 新增列：默认值走应用层回填，避免锁表

## 分库分表路由

当采用分库/分表模式：

- 路由键：`TenantId`（或 `TenantId + CreatedAt` 冷热分离）
- 中间件：ShardingCore / 自研 `ITenantConnectionResolver`
- 路由表缓存：启动期加载 + 变更事件失效
- 跨片查询：禁用；改走聚合层 / 读模型

## 审计表与归档

- `SysAuditLog` 按月分区，热数据 3 个月在库，冷数据归档对象存储
- 审计表**只追加**，不允许 `UPDATE/DELETE`（用触发器或权限保证）
- 保留期满足合规最低要求（通常 ≥ 6 个月）

## DBA 评审清单

```
- [ ] 所有新表含 TenantId 且为索引首列
- [ ] 唯一约束已带 TenantId
- [ ] 软删索引过滤已加
- [ ] 外键不跨租户
- [ ] 迁移脚本含 up/down/verify
- [ ] 批量 DML 带显式 WHERE TenantId
- [ ] 大表变更走在线 DDL 工具
- [ ] 权限/审计表已规划分区与归档
- [ ] 查询计划审核：无不带 TenantId 的 seek
- [ ] 连接池按租户或分库维度隔离（如采用）
```

## 常见 DB 反模式

- ❌ 历史表没有 `TenantId`，靠外键间接关联（漏 JOIN 就越权）
- ❌ 为了节省索引在 `TenantId` 列用低基数索引（应作为复合索引首列）
- ❌ 同一 `TenantId` 下唯一但未建唯一约束（业务层校验失效即脏数据）
- ❌ 迁移脚本用 ORM 在应用启动时执行（不可回滚、审计缺失）
- ❌ 软删除用 `DELETE`；硬删除后审计链断裂
