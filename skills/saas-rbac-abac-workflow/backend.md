# 后端开发交付物规范（ASP.NET Core）

后端是 RBAC + ABAC 的执行中枢。所有权限判定必须在后端完成，前端仅负责"渲染与体验"。

## 核心职责

1. 落地 `ITenantResolver` / `ITenantContext`（每请求范围）
2. 实现权限中间件、授权策略处理器、数据范围过滤器
3. 实现 ABAC 条件解释器与属性提供器
4. 落地字段级安全（FLS）序列化管道
5. 提供权限元数据接口（菜单、按钮、数据范围下发）
6. 审计日志钩子

## 目录分层约定

```
Domain/               # 实体、枚举、领域服务
Application/          # 用例、DTO、权限码常量、策略名
Infrastructure/       # EF DbContext、缓存、审计、ABAC 解释器
HttpApi/              # Controller / MinimalApi、全局过滤器
```

## 租户上下文

```csharp
public interface ITenantContext
{
    long TenantId { get; }
    bool IsPlatform { get; }     // 平台超管
    long UserId { get; }
    IReadOnlyList<string> Permissions { get; }
    DataScope DataScope { get; }
}
```

解析优先级（`ITenantResolver`）：
1. JWT claim `tid`（首选，可信）
2. 服务间调用 Header `X-Tenant-Id`（需内网 mTLS）
3. ❌ 禁止从 URL / Query 取

**每请求一次解析**，注入 `Scoped` 生命周期。

## 权限码常量

```csharp
public static class Permissions
{
    public static class Saas
    {
        public static class User
        {
            public const string Read   = "saas:user:read";
            public const string Create = "saas:user:create";
            public const string Update = "saas:user:update";
            public const string Delete = "saas:user:delete";
        }
    }
}
```

启动期通过 `IPermissionProvider` 反射收集并 Seed 到 `SysPermission` 表。

## Controller 端点模板

```csharp
[ApiController]
[Route("api/saas/users")]
[Authorize] // AuthN
public class UserController : ControllerBase
{
    [HttpGet]
    [HasPermission(Permissions.Saas.User.Read)]     // RBAC
    [DataScope(ResourceType = "user")]              // 数据范围
    public async Task<PagedResult<UserDto>> List([FromQuery] UserQuery q)
        => await _service.ListAsync(q);

    [HttpPost]
    [HasPermission(Permissions.Saas.User.Create)]
    [AuditAction("CreateUser", Sensitivity = AuditSensitivity.High)]
    public async Task<UserDto> Create([FromBody] CreateUserInput input)
        => await _service.CreateAsync(input);
}
```

**`[HasPermission]` 实现要点**：
- 基于 `IAuthorizationRequirement` + `AuthorizationHandler`
- 命中缓存 `perm:{tenantId}:{userId}` → 判定 → 失败返回 403（而非 401）
- 审计"授权失败"本身也是敏感事件

## 数据范围过滤

```csharp
public class DataScopeFilter<TEntity> where TEntity : TenantEntity
{
    public IQueryable<TEntity> Apply(IQueryable<TEntity> q, ITenantContext ctx)
    {
        return ctx.DataScope.Kind switch
        {
            DataScopeKind.Self         => q.Where(x => x.CreatedBy == ctx.UserId),
            DataScopeKind.Dept         => q.Where(x => ctx.DataScope.DeptIds.Contains(x.DeptId)),
            DataScopeKind.DeptAndChild => q.Where(x => ctx.DataScope.DeptIds.Contains(x.DeptId)),
            DataScopeKind.Tenant       => q,                // 全局过滤器已限 TenantId
            DataScopeKind.Custom       => q.Where(x => ctx.DataScope.DeptIds.Contains(x.DeptId)),
            DataScopeKind.Global       => q.IgnoreQueryFilters(),  // 仅平台超管
            _ => q.Where(x => false),
        };
    }
}
```

规则：
- 列表接口**必须**调用 `DataScopeFilter`；通过静态分析/代码审查强制
- `IgnoreQueryFilters` 仅在 `ctx.IsPlatform` 为真时允许

## ABAC 策略处理器

```csharp
public class AbacHandler : AuthorizationHandler<AbacRequirement, object>
{
    protected override async Task HandleRequirementAsync(
        AuthorizationHandlerContext ctx, AbacRequirement req, object resource)
    {
        var attrs = await _attrProvider.BuildAsync(ctx.User, resource, HttpCtx());
        var decision = _evaluator.Evaluate(req.PolicyId, attrs);  // allow / deny / notApplicable
        if (decision == Decision.Allow) ctx.Succeed(req);
        else if (decision == Decision.Deny) ctx.Fail();           // deny 压 allow
    }
}
```

**条件解释器** 仅支持白名单算子（见 SKILL.md 附录）。表达式解析结果缓存至 `abac:{tenantId}:{policyId}:v{n}`。

## 字段级安全（FLS）管道

挂在 JSON 序列化阶段：

```csharp
services.AddControllers(o => o.Filters.Add<FlsOutputFilter>())
        .AddJsonOptions(o => o.JsonSerializerOptions
            .Converters.Add(new FlsConverterFactory(fieldMasker)));
```

`IFieldMasker` 根据 `SysFieldLevelSecurity` 规则对字段执行：

- `FullMask` → 全部替换为 `****`
- `PartialMask` → 保留首尾各 N 位
- `Hash` → SHA-256 前 8 位
- `Encrypted` → 返回 `enc:...`（仅具备解密权限的视图层使用）
- `Deny` → 从返回体中**删除该字段**（JsonIgnore 动态）

导出接口（Excel/CSV）单独走 `FlsExportFilter`，禁止绕过。

## 审计

```csharp
[AuditAction("DeleteUser", Sensitivity = AuditSensitivity.High)]
public Task Delete(long id) { ... }
```

拦截器在成功/失败后写入 `IAuditSink`（异步 + 批量），字段：

```
TenantId, UserId, Action, Resource, ResourceId, Success,
ClientIp, UserAgent, CorrelationId, Before(json), After(json), OccurredAt
```

规则：
- 审计写入**绝不阻塞主流程**（失败降级到本地文件 + 告警）
- 审计不可被业务模块反向查询（独立只读视图）

## 缓存与失效

```csharp
// 权限变更时
await _bus.Publish(new PermissionChangedEvent(tenantId, userId));

// 订阅端
public Task Handle(PermissionChangedEvent e)
    => _cache.RemoveAsync($"perm:{e.TenantId}:{e.UserId}");
```

- L1：进程内 `IMemoryCache`，TTL 60s
- L2：`IDistributedCache`（Redis），TTL 10 分钟
- 变更广播：Redis Pub/Sub 或消息总线

## 后端评审清单

```
- [ ] 所有写操作均经过 [HasPermission] 或等价策略
- [ ] 所有列表查询均走 DataScopeFilter
- [ ] 敏感字段走 FLS，未在 Controller 手工裁剪
- [ ] TenantId 来源仅 JWT / 内部可信 Header
- [ ] 权限缓存失效事件已发布 + 订阅
- [ ] ABAC 表达式在白名单算子内
- [ ] 审计拦截器覆盖所有写操作与敏感读
- [ ] 平台超管路径显式 IgnoreQueryFilters，并单独审计
- [ ] 单测覆盖：无权限 / 跨租户 / 数据范围三类分支
```

## 常见后端反模式

- ❌ 在 Service 里用 `_db.Set<T>()` 绕过 DbContext 的全局过滤器
- ❌ 在 `OnActionExecuting` 里检查权限（时机太晚，已经过模型绑定）
- ❌ 把权限判定写进 `AutoMapper` 的 `ResolveUsing`（难测、难审计）
- ❌ 用 `Task.Run` 异步写审计却未捕获异常
- ❌ JWT 未签名 / 不含 `tid` / 过期时间过长（> 2h）
- ❌ 把整个 `Role` 对象写进 JWT 而非权限码数组（Token 臃肿且难失效）
