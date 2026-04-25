---
name: xihan-basicapp-fullstack-rearchitect
description: XiHanFun 全栈重构规范 — 实体驱动重构、多租户/RBAC/ABAC/FLS/动态API/软删除、仓储服务重建、前端页面重建。触发于 XiHan.Framework 底层修复、XiHan.BasicApp.Saas 领域重构、frontend/src 页面重建等任务。
---

# XiHan BasicApp 全栈重构规范

## 一、项目边界

将仓库视为完整开源产品族：

- `XiHan.Framework`：底层框架（47 个模块），通用基础设施。
- `XiHan.BasicApp`：基于 Framework 的前后端基础应用。
- `XiHan.BasicApp.Saas/Domain/Entities`：SaaS 实体模型，业务重构的主要事实源。
- `XiHan.BasicApp/frontend/packages`：前端基础框架，原则上不改。
- `XiHan.BasicApp/frontend/src`：前端业务页面、API、store、路由的主要修改面。
- `XiHan.Framework.Web.Api/DynamicApi`：动态 API 生成能力，禁止新增 Controller。

## 二、项目结构

```
XiHanFun/
├── XiHan.Framework/              # 底层框架（47 个模块）
│   └── framework/src/
│       ├── XiHan.Framework.Core/
│       ├── XiHan.Framework.Domain/         # DDD 基础（EntityBase, AggregateRootBase, 仓储接口）
│       ├── XiHan.Framework.Data/           # SqlSugar ORM + 实体基类 + 全局过滤器
│       ├── XiHan.Framework.Application/    # 应用服务基类（CrudApplicationServiceBase）
│       ├── XiHan.Framework.Web.Api/        # DynamicApi 动态接口生成
│       ├── XiHan.Framework.MultiTenancy/   # 多租户抽象
│       ├── XiHan.Framework.Authorization/  # 授权基础
│       └── ...（其余 40+ 模块）
│
└── XiHan.BasicApp/               # 前后端基础应用
    ├── backend/src/
    │   ├── framework/
    │   │   ├── XiHan.BasicApp.Core/        # 共享内核（BasicAppEntity 基类）
    │   │   └── XiHan.BasicApp.Web.Core/    # Web 层框架
    │   ├── modules/
    │   │   ├── XiHan.BasicApp.Saas/        # 主业务模块（巨模块，需按领域拆分）
    │   │   └── XiHan.BasicApp.CodeGeneration/
    │   └── main/
    │       └── XiHan.BasicApp.WebHost/     # 启动入口
    └── frontend/
        ├── packages/                        # 基础框架包（尽量不动）
        └── src/                             # 应用层（重构目标）
```

## 三、技术栈

| 层 | 技术 |
|---|---|
| 后端运行时 | .NET 10 |
| ORM | SqlSugar（多租户、全局过滤器、分表） |
| 接口生成 | DynamicApi（无 Controller，AppService 自动映射 HTTP 端点） |
| 前端框架 | Vue 3.5 + TypeScript 5.9 |
| UI 库 | Naive UI 2.44 |
| 状态管理 | Pinia 2.3 |
| 表格 | VxeTable 4.18 |
| 构建工具 | Vite 6.4 |
| 包管理 | pnpm 10.4 |
| 实时通信 | SignalR |

## 四、重构执行顺序（严格自底向上）

按以下顺序推进，不要用上层补丁掩盖下层设计问题。如果在上层工作中发现下层问题，先修下层，或明确记录为阻塞风险后再继续。

```
第 1 层：底层框架优化（XiHan.Framework）
  → 实体基类补全、全局过滤器一致性、值对象支持
第 2 层：应用内核优化（XiHan.BasicApp.Core）
  → BasicAppEntity 基类调整、公共值对象定义
第 3 层：领域实体重构（Domain/Entities）
  → 模块拆分、聚合根精简、枚举语义化、导航关系修正
第 4 层：领域层完善（Domain/）
  → 仓储接口、领域服务、领域事件、规约
第 5 层：基础设施层（Infrastructure/）
  → 仓储实现、认证授权、多租户适配
第 6 层：应用服务层（Application/）
  → AppService、QueryService、DTO、映射、缓存
第 7 层：前端 API 层（frontend/src/api/）
  → 接口定义、类型、请求封装
第 8 层：前端页面层（frontend/src/views/）
  → 页面组件、路由、状态管理
```

## 五、AI 工作流程

### 5.1 优先读取

每次触发后，按任务范围精准读取，不要批量塞上下文：

1. 相关实体、Partial、Aggregate、Enum
2. 相关框架抽象：实体基类、租户上下文、数据过滤器、仓储、UoW、动态 API、缓存、设置
3. 相关 `frontend/src` API、类型、store、视图、路由

优先用 `rg` / `rg --files` 检索。

### 5.2 框架修改规则

只有确属通用基础设施时，才修改 `XiHan.Framework`：

- 实体基类语义、多租户上下文、软删除/租户数据过滤器
- 仓储基类、工作单元与领域事件
- 动态 API 生成、缓存和设置抽象、安全基础能力

不要把 SaaS 业务概念下沉到框架。角色、权限、租户成员、套餐、FLS、SaaS 种子模板属于 `XiHan.BasicApp.Saas`。框架改动必须保持通用、可复用、可解释。

### 5.3 交付格式

较大任务完成时，输出简洁追踪信息：解决的需求或设计问题、修改文件、行为变化、验证结果、剩余风险。

架构决策变化时同步更新文档。

## 六、后端架构规范

### 6.1 实体基类使用规则

框架提供的基类继承链（`XiHan.Framework.Data.SqlSugar`）：

```
SugarMultiTenantEntity<long>              → BasicAppEntity             # 仅 TenantId + Id
SugarMultiTenantCreationEntity<long>      → BasicAppCreationEntity     # + CreatedTime/Id/By
SugarMultiTenantModificationEntity<long>  → BasicAppModificationEntity # + ModifiedTime/Id/By
SugarMultiTenantDeletionEntity<long>      → BasicAppDeletionEntity     # + IsDeleted + DeletedTime/Id/By
SugarMultiTenantFullAuditedEntity<long>   → BasicAppFullAuditedEntity  # 创建 + 修改 + 软删除
SugarMultiTenantAggregateRoot<long>       → BasicAppAggregateRoot      # 完整审计 + 领域事件
```

选择原则：

| 场景 | 基类 |
|---|---|
| 关联表（多对多中间表） | `BasicAppCreationEntity` |
| 日志/历史记录（只写不改不删） | `BasicAppCreationEntity` + `ISplitTableEntity` |
| 普通业务实体（需要审计） | `BasicAppFullAuditedEntity` |
| 真正的聚合根（有不变量、发领域事件） | `BasicAppAggregateRoot` |

聚合根判定标准（必须同时满足）：
1. 拥有需要保护的业务不变量（invariant）
2. 是事务一致性边界
3. 需要发布领域事件
4. 不满足以上条件的实体，禁止使用 `BasicAppAggregateRoot`

### 6.2 实体设计规范

主键统一使用 `long`（雪花算法），由框架 `Aop.DataExecuting` 自动填充。

| 字段 | 类型 | 说明 |
|------|------|------|
| `BasicId` | `long` | 主键（雪花算法） |
| `RowVersion` | `long` | 乐观锁控制 |
| `TenantId` | `long` | 租户隔离（0=平台） |
| `CreatedTime` | `DateTimeOffset` | UTC 时间戳 |
| `IsDeleted` | `bool` | 软删除标志 |

TenantId 约定：
- `TenantId = 0` 为平台级（全局）数据，业务租户从 1 开始
- 禁止用 `TenantId IS NULL` 表达平台语义
- 查询自动合并全局 + 当前租户：`WHERE TenantId IN (0, {currentTenantId})`
- `SysTenant` 是平台元数据，不能被业务租户过滤误隐藏
- 租户数据写入时 `TenantId` 由框架 AOP 自动填充，禁止手动赋值

实体文件拆分（Partial Class）：
```
Domain/Entities/
  SysUser.cs              # 数据属性定义
Domain/Entities/Expands/
  SysUser.Expand.cs           # SqlSugar 索引/表属性等扩展配置
Domain/Entities/Aggregates/
  SysUser.Aggregate.cs    # 聚合根行为方法（仅聚合根需要）
Domain/Entities/Enums/
  SysUser.Enum.cs    # 实体所需要的枚举
```

枚举语义化要求：
- 禁止用 `YesOrNo` 表示业务状态
- 每个业务场景定义独立枚举：`EnableStatus`、`ValidityStatus`、`InviteStatus` 等
- 枚举值必须有 `[Description("中文描述")]` 特性

索引约束标准：
```csharp
[SugarIndex("IX_{table}_TeId_CrTi", nameof(TenantId), OrderByType.Asc, nameof(CreatedTime), OrderByType.Desc)]
[SugarIndex("UX_{table}_TeId_RoCo", nameof(TenantId), nameof(RoleCode), true)] // 唯一索引
```
- 每个实体必须有 `(TenantId, CreatedTime)` 组合索引
- 业务唯一键必须有 `TenantId` 前缀
- 外键字段必须建立普通索引加速 JOIN

### 6.3 多租户隔离机制

当前实现：
- 通过 SqlSugar 全局查询过滤器自动附加 `WHERE TenantId IN (0, {currentTenantId})`
- 支持跨租户切换：`ICurrentTenant.Change(tenantId)`
- `EnableAutoUpdateQueryFilter = true` 和 `EnableAutoDeleteQueryFilter = true` 确保 UPDATE/DELETE 也受过滤器约束

红线规范：
- 禁止将 `TenantId` 放入 URL 或 Query 参数参与鉴权
- 禁止在 Controller 中手写 `if (user.TenantId != entity.TenantId)`
- 禁止遗漏 `TenantId` 的批量更新脚本（必须有 WHERE 条件）
- 禁止在业务代码中手动拼接 `TenantId` 或 `IsDeleted` 条件
- 跨租户操作仅限平台管理员，使用 `CreateNoTenantQueryable()` 并记录审计日志

逃逸开关：
- `CreateNoTenantQueryable()` — 跨租户查询
- `CreateWithDeletedQueryable()` — 审计/恢复场景清除软删除过滤器
- `IgnoreMultiTenancyAttribute` — 平台级管理功能忽略租户过滤

### 6.4 软删除与审计

- `ISoftDelete` → `IDeletionEntity` → `IDeletionEntity<TKey>` 接口链
- SqlSugar `QueryFilter<ISoftDelete>()` 自动附加 `WHERE IsDeleted = false`
- 审计字段自动注入：`CreatedTime`, `ModifiedTime`, `DeletedTime` 等
- 所有支持软删除的实体必须实现 `ISoftDelete`（`BasicAppDeletionEntity` 及以上自动满足）
- 恢复删除记录使用 `RestoreAsync()`，不要直接 UPDATE

授权事实优先使用生命周期字段，不要只硬删：授权/撤销状态、生效/失效时间、操作人、原因、审计事件。

### 6.6 值对象定义

在 `Domain/ValueObjects/` 中定义公共值对象，各实体通过 SqlSugar `[SugarColumn(IsJson = true)]` 或展开字段方式复用：

```csharp
public record ClientInfo(string? Ip, string? Location, string? UserAgent, string? Browser, string? Os);
public record EffectivePeriod(DateTime? EffectiveTime, DateTime? ExpirationTime);
public record BusinessReference(string? BusinessType, long? BusinessId);
public record DeviceInfo(string? DeviceType, string? DeviceName, string? DeviceId, string? Os, string? Browser);
```

### 6.7 DynamicApi 规范

禁止创建 Controller。所有 HTTP 端点通过 AppService + `[DynamicApi]` 自动生成。

```csharp
[DynamicApi(Group = "BasicApp.Saas", GroupName = "系统Saas服务")]
[Authorize]
public class UserAppService
    : CrudApplicationServiceBase<SysUser, UserDto, long, UserCreateDto, UserUpdateDto, BasicAppPRDto>,
      IUserAppService
```

HTTP 方法映射（由 `DefaultDynamicApiConvention` 自动推断）：

| 方法名前缀 | HTTP 方法 |
|---|---|
| `Get*`, `Find*`, `Query*`, `Search*` | GET |
| `Create*`, `Add*`, `Insert*` | POST |
| `Update*`, `Edit*`, `Modify*` | PUT |
| `Delete*`, `Remove*` | DELETE |

路由生成规则：`api/{ControllerName}/{ActionName}`
- ControllerName = 类名去掉 `AppService`/`ApplicationService`/`Service` 后缀
- ActionName = 方法名去掉 HTTP 前缀和 `Async` 后缀

暴露接口前必须：
1. 查看 `XiHan.Framework.Web.Api/DynamicApi` 的本地约定
2. 使用可被动态 API 稳定识别的应用服务命名与方法签名
3. 不把前端传入的 `TenantId` 作为鉴权租户
4. 授权放在后端服务，不依赖前端隐藏
5. 扫描确认没有新增 Controller

### 6.8 仓储规范

接口层（Domain/Repositories/）：
```csharp
public interface IUserRepository : IAggregateRootRepository<SysUser, long>
{
    Task<SysUser?> GetByUserNameAsync(string userName);
}
```

实现层（Infrastructure/Repositories/）：
```csharp
public class UserRepository : SqlSugarAggregateRepository<SysUser, long>, IUserRepository
{
    public UserRepository(ISqlSugarClientResolver clientResolver, IUnitOfWorkManager unitOfWorkManager)
        : base(clientResolver, unitOfWorkManager) { }
}
```

分表日志仓储继承 `SqlSugarSplitTableRepository<TEntity, long>`，使用 `ISplitTableEntity` 标记。

### 6.9 应用服务分层

```
Application/
├── AppServices/                    # 写操作服务接口 + 实现（[DynamicApi] 标记）
├── QueryServices/                  # 读操作服务（CQRS 读侧，可带缓存）
├── Dtos/                           # 数据传输对象（按领域分子目录）
├── Mappers/                        # 对象映射配置
├── EventHandlers/                  # 领域事件处理器
└── Caching/                        # 缓存策略
```

后端职责边界：
- Domain Entities：实体状态与领域不变式
- Domain Services：跨聚合业务规则
- Repositories：聚合持久化和稳定数据访问，不做页面拼装
- Query Services / Read Models：列表、树、详情等读模型投影
- Application Services：用例编排、事务边界、权限入口
- Internal Services：授权、租户成员、数据范围、FLS、缓存失效、种子编排

如果旧服务、旧仓储、旧 DTO、旧页面与实体模型冲突，可以直接重构替换，不做兼容性小补丁。

## 七、RBAC + ABAC 权限模型

### 7.1 权限码命名规范

```
{模块}:{资源}:动作[:子动作]
示例:
  saas:user:read         - 查看用户列表
  saas:user:create       - 创建用户
  crm:order:export:sensitive - 导出敏感订单
```

权限标注在 Service 方法上：`[PermissionAuthorize("saas:user:read")]`
ABAC 策略在 Service 方法上：`[AbacAuthorize("order.read.dept-scope")]`

### 7.2 权限模型设计

```csharp
// 角色-权限关联支持数据范围
public class SysRolePermission : BasicAppCreationEntity
{
    public long RoleId { get; set; }
    public long PermissionId { get; set; }
    public PermissionAction PermissionAction { get; set; } // Grant/Deny
    public DataPermissionScope DataScope { get; set; }
    public List<DataScopeFilter>? Filters { get; set; } // JSON
    public DateTimeOffset? EffectiveTime { get; set; }
    public DateTimeOffset? ExpirationTime { get; set; }
}

// 权限表
public class SysPermission : BasicAppAggregateRoot
{
    public string ModuleCode { get; set; }      // saas/crm/billing
    public string PermissionCode { get; set; }  // user:read
    public string PermissionName { get; set; }
    public int Priority { get; set; }           // 冲突时优先级
    // IsGlobal 改为计算属性：public bool IsGlobal => TenantId == 0;
}
```

### 7.3 ABAC 策略表达式

```json
{
  "policyId": "order.read.dept-scope",
  "effect": "allow",
  "subject": { "hasPermission": "crm:order:read" },
  "condition": {
    "all": [
      { "eq": ["resource.tenantId", "subject.tenantId"] },
      { "in": ["resource.deptId", "subject.dataScope.deptIds"] }
    ]
  }
}
```

支持的操作符：`eq`, `ne`, `in`, `notIn`, `gt`, `gte`, `lt`, `lte`, `all`, `any`, `not`

属性命名空间：
- `subject.*` — 用户/角色信息（如 `subject.department.id`）
- `resource.*` — 资源数据（如 `resource.status`）
- `environment.*` — 环境上下文（如 `environment.hour`, `environment.ip`）

### 7.4 字段脱敏（FLS）

```csharp
public partial class SysFieldLevelSecurity : BasicAppFullAuditedEntity
{
    public FieldSecurityTargetType TargetType { get; set; } // Role/User/Permission
    public long TargetId { get; set; }
    public long ResourceId { get; set; }
    public string FieldName { get; set; }
    public bool IsReadable { get; set; }
    public bool IsEditable { get; set; }
    public FieldMaskStrategy MaskStrategy { get; set; } // None/PartialMask/Hash/Redact
    public string? MaskPattern { get; set; } // {"keepLeft":3,"keepRight":4}
    public int Priority { get; set; }
}
```

实施要求：
- 仅在序列化出口和导出出口执行脱敏
- 禁止前端二次裁剪敏感数据
- 脱敏配置变更必须审计

## 八、前端架构规范

### 8.1 目录结构

```
frontend/src/
├── api/
│   ├── request.ts              # RequestClient 实例
│   ├── base.ts                 # useBaseApi() CRUD 工厂
│   ├── helpers.ts              # 通用工具函数
│   └── modules/                # 按后端模块对应，一个文件一个领域
│       ├── identity/           # user.ts + types.ts
│       ├── authorization/      # role.ts, permission.ts, menu.ts + types.ts
│       ├── organization/       # department.ts
│       ├── tenant/             # tenant.ts
│       └── ...（与后端模块一一对应）
├── router/routes/index.ts      # 静态路由
├── views/                      # 按领域分目录（identity/, authorization/, tenant/...）
└── types/                      # 自动生成的类型声明
```

### 8.2 API 层规范

```typescript
// api/modules/identity/types.ts — 类型定义与后端 DTO 一一对应
export interface UserDto {
  id: number;
  userName: string;
  realName?: string;
}

// api/modules/identity/user.ts — API 函数
const baseApi = useBaseApi('User');
export const userApi = {
  page: (params: PageRequest) => baseApi.page<UserDto>(params),
  detail: (id: number) => baseApi.detail<UserDto>(id),
  create: (data: UserCreateDto) => baseApi.create<UserDto>(data),
  update: (data: UserUpdateDto) => baseApi.update<UserDto>(data),
  delete: (id: number) => baseApi.delete(id),
};
```

### 8.3 页面组件规范

每个管理页面遵循统一结构：
```
views/identity/user/
├── index.vue           # 主页面（列表 + 搜索）
├── components/
│   ├── UserForm.vue    # 新增/编辑表单（NaiveUI NForm）
│   ├── UserDetail.vue  # 详情抽屉/弹窗
│   └── UserSearch.vue  # 搜索条件面板
```

表格统一使用 VxeTable，表单使用 Naive UI `NForm`。

### 8.4 packages/ 修改原则

`frontend/packages/` 为基础框架层，原则上不修改。例外情况：

- `packages/types/` 公共类型需要与后端 DTO 对齐
- `packages/stores/` 的 `useAccessStore`/`useUserStore` 需要适配新权限模型
- `packages/router/` 动态路由逻辑需要适配新菜单结构
- 发现明确的 bug 或设计缺陷

修改 packages 时必须说明原因并确保向后兼容。

### 8.5 前端重构规则

- API 类型必须匹配后端 DTO，删除旧兼容字段，不做静默映射
- 租户、角色、权限、菜单、部门、数据范围、ABAC、FLS 语义必须与后端一致
- 前端只做展示和交互，不承担真正授权

## 九、安全规范

### 9.1 认证与授权

- 所有 AppService 默认 `[Authorize]`，仅登录/注册等匿名接口使用 `[AllowAnonymous]`
- 权限控制使用 `[PermissionAuthorize("module:operation")]` 特性
- 前端路由守卫 + 后端权限校验双重保障，不可仅依赖前端
- JWT Token 存储在 `LocalStorage`，刷新令牌机制已内置于 `RequestClient`

### 9.2 数据安全

- 密码字段（`Password`、`ClientSecret`、`TwoFactorSecret`、`SecurityStamp`）必须标记 `[JsonIgnore]`
- Token 字段（`AccessToken`、`RefreshToken`、`Code`）必须标记 `[JsonIgnore]`
- 连接字符串（`ConnectionString`）必须标记 `[JsonIgnore]`
- 敏感字段禁止出现在 DTO 的响应类型中
- 禁止记录原始密码、Token、Secret、连接串、Authorization、Cookie、未脱敏请求体和响应体

### 9.3 Session/Token 一致性

- Session 撤销必须通过领域事件级联撤销所有关联 Token
- Token 刷新时必须验证关联 Session 的有效性
- 多设备登录限制在 `SysUserSecurity.MaxLoginDevices` 中配置

### 9.4 审计日志覆盖

以下操作必须记录 `SysAuditLog`：
- 权限授予/撤销（包括 FLS 字段脱敏配置变更）
- 敏感数据访问（身份证号、薪资、手机号等）
- 跨租户操作（`TenantId != CurrentTenantId`）
- 管理员账号操作（SuperAdmin, Owner 权限）

### 9.5 安全参考

- 多租户/数据过滤：显式租户上下文、数据过滤器、平台记录合并
- 认证：ASP.NET Core Identity / OpenIddict 类实践
- 授权：RBAC、deny-overrides、显式租户成员、最小权限
- API 安全：OWASP ASVS / OWASP Cheat Sheets
- 密钥：可验证场景存 Hash，必须恢复场景才加密

## 十、已知设计问题与修复方案

### 10.1 聚合根降级清单

| 实体 | 降级为 | 原因 |
|---|---|---|
| `SysFieldLevelSecurity` | `BasicAppFullAuditedEntity` | 无独立不变量 |
| `SysConstraintRule` | `BasicAppFullAuditedEntity` | RBAC 策略子实体 |
| `SysNotification` | `BasicAppFullAuditedEntity` | 简单消息实体 |
| `SysEmail` | `BasicAppFullAuditedEntity` | 简单消息实体 |
| `SysSms` | `BasicAppFullAuditedEntity` | 简单消息实体 |
| `SysFile` | `BasicAppFullAuditedEntity` | 文件元数据无复杂不变量 |
| `SysDict` | `BasicAppFullAuditedEntity` | 简单 CRUD |
| `SysConfig` | `BasicAppFullAuditedEntity` | 简单 CRUD |
| `SysUserSession` | `BasicAppFullAuditedEntity` | 会话记录无领域事件需求 |

保留为聚合根：`SysUser`、`SysTenant`、`SysRole`、`SysPermission`、`SysResource`、`SysOperation`、`SysDepartment`、`SysOAuthApp`、`SysTask`、`SysReview`。

### 10.2 SysPermission 灵活化

将 `ResourceId` 和 `OperationId` 改为可空 `long?`，新增 `PermissionType` 枚举：

```csharp
public enum PermissionType
{
    [Description("资源操作权限")] ResourceBased = 0,
    [Description("功能权限")] Functional = 1,
    [Description("数据范围权限")] DataScope = 2,
}
```

### 10.3 SysUser 导航属性精简

仅保留核心导航：`Security`（1:1）、`UserRoles`、`UserPermissions`、`UserDepartments`。
移除所有日志、通知、邮件、短信、文件、OAuth、统计等导航属性，通过各自领域的仓储按需查询。

### 10.4 其他修复项

- YesOrNo 枚举替换：逐实体替换为语义明确的枚举（`EnableStatus`、`ValidityStatus` 等）
- 日志实体公共字段提取：在 `Domain/ValueObjects/` 定义 `RequestContext` 值对象
- SysFileStorage 精简：移除 `SignedUrl`、`SignedUrlExpiresAt`（运行时生成）、`StorageDirectory`（由 `StoragePath` 推导）、`Endpoint`/`CustomDomain`（从配置关联获取）
- SysTenant 补充导航：添加 `TenantUsers` 导航属性指向 `SysTenantUser`



## 十一、编码风格

### 11.1 C# 后端

- 使用 C# 最新语法特性（file-scoped namespace、primary constructor、record 等）
- 实体属性使用 `virtual` 修饰（SqlSugar 延迟加载需要）
- 异步方法统一 `Async` 后缀
- 仓储方法命名：`GetByXxxAsync`、`FindByXxxAsync`、`ExistsXxxAsync`
- 禁止在实体中写业务逻辑，聚合根行为方法放在 `Aggregates/` 分部类中

### 11.2 TypeScript 前端

- 使用 Composition API（`<script setup lang="ts">`）
- 类型定义与 API 模块同目录
- 组件命名 PascalCase，文件名 PascalCase
- 使用 Naive UI 组件，禁止引入其他 UI 库
- 表格统一 VxeTable，表单统一 NForm

### 11.3 通用

- 必要注释，summary 必须
- 不做过度抽象，三次重复再提取
- 不做向后兼容 hack，直接重构
- 不留 TODO/FIXME，当场解决或记录为 issue

## 十二、验证门禁

先跑最小有效验证，再扩大范围。

后端：
- `dotnet build` 目标项目或 SaaS 模块
- 存在测试时跑定向测试
- 租户/API 架构变更后扫描：`class .*Controller`、`TenantId IS NULL`、`PlatformTenantId = 1`、`TenantId == null`

前端：
- `pnpm type-check`
- `pnpm build`
- 仓库已有 lint 规则时跑定向 lint

跨层：
- 租户筛选与软删除筛选在框架和项目侧一致
- 接口通过动态 API 暴露，而不是 Controller
- 角色、权限、菜单、部门、FLS 变更后缓存失效正确
- 安全敏感改动必须覆盖负权限和跨租户场景

如果验证被环境问题阻塞，说明具体阻塞，并补充代码事实扫描结果。

## 十三、修改检查清单

每次修改前确认：

- [ ] 实体基类选择是否正确（参考 6.1）
- [ ] TenantId 约定是否遵守（参考 6.2）
- [ ] 全局过滤器是否覆盖（参考 6.3/6.4）
- [ ] 敏感字段是否标记 `[JsonIgnore]`（参考 9.2）
- [ ] 是否有 Controller（禁止，参考 6.7）
- [ ] 前端类型是否与后端 DTO 对齐（参考 8.2）
- [ ] 枚举是否语义明确（参考 6.2）
- [ ] 导航属性是否精简（参考 10.3）
- [ ] 新增权限码已在 PermissionSeed 登记
- [ ] 菜单/按钮/数据范围三层权限齐备
- [ ] 敏感字段走 FLS 脱敏
- [ ] 审计日志覆盖授权变更 + 敏感读写
- [ ] 至少包含：正权限/负权限/跨租户越权三类测试
- [ ] 灰度开关与回滚脚本已就绪

## 十四、最佳实践参考

| 领域 | 参考项目/规范 |
|---|---|
| DDD 聚合设计 | Vaughn Vernon《实现领域驱动设计》、ABP Framework 聚合根设计 |
| RBAC/ABAC | NIST RBAC 标准（INCITS 359-2012）、Casbin 策略模型 |
| 多租户 | ABP Framework 多租户模块、SaaS Toolkit 模式 |
| 前端状态管理 | Pinia 官方最佳实践 |
| API 设计 | Microsoft REST API Guidelines |
| 安全 | OWASP Top 10、OWASP ASVS |
| 日志 | OpenTelemetry 语义约定 |
| OAuth 2.0 | RFC 6749、RFC 7636（PKCE） |
| 动态 API | FastAPI / Swashbuckle 元数据驱动接口生成 |
| 字段脱敏 | GDPR 合规指南 FLS (Field Level Security) |
| 软删除 | EF Core Global Queries 透明过滤模式 |
