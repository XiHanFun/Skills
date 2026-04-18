# 测试工程师交付物规范

测试的首要目标是**证明隔离与授权不可被绕过**，其次才是功能正确。

## 核心职责

1. 维护权限矩阵测试用例与回归集
2. 多租户隔离自动化测试
3. 字段级安全（FLS）出口测试
4. 审计日志正确性验证
5. 性能与并发下的授权一致性测试

## 测试分层

| 层级 | 目标 | 工具 |
|------|------|------|
| 单元测试 | 策略处理器、数据范围过滤器、FLS 转换器 | xUnit / Vitest |
| 集成测试 | Controller + DB 真表（含全局过滤器） | WebApplicationFactory + Testcontainers |
| E2E | 真实登录 → UI 跳转 → 数据隔离 | Playwright |
| 合约测试 | 前后端权限码/返回 DTO 一致 | Pact / OpenAPI diff |
| 性能测试 | 高并发下授权缓存与判定稳定 | k6 / NBomber |

## 权限矩阵用例模板

对每个受保护端点填写：

```markdown
### 端点：GET /api/saas/users/{id}

| # | Subject 角色/权限 | Resource 归属 | 数据范围 | ABAC 条件 | 期望 | 说明 |
|---|-------------------|---------------|----------|-----------|------|------|
| 1 | 无 saas:user:read | 同租户 | - | - | 403 | RBAC |
| 2 | 有权限，跨租户 | 他租户 | - | - | 403 | 租户隔离 |
| 3 | 有权限，同租户 | 本人 | Self | - | 200 | 正向 |
| 4 | 有权限，同租户 | 他人同部门 | Self | - | 403 | 数据范围 |
| 5 | 有权限，同租户 | 他人同部门 | Dept | - | 200 | 数据范围 |
| 6 | 有权限，跨部门 | 他部门 | DeptAndChild | - | 200/403 | 看部门树 |
| 7 | 有权限 | 同租户 | Tenant | status=Archived 且策略禁访问已归档 | 403 | ABAC |
| 8 | 平台超管 | 任意租户 | Global | - | 200 | 超管 |
| 9 | 有权限 | 同租户 | Tenant | - | 200 字段手机号=`138****5678` | FLS |
| 10 | 有权限（Deny 字段） | 同租户 | Tenant | - | 200 响应体无 idCard 字段 | FLS Deny |
```

每个新增端点**至少**包含用例 1、2、9 三条。

## 隔离回归测试套件

```csharp
[Theory]
[InlineData("TenantA-UserA", "TenantB-Order1", HttpStatusCode.Forbidden)]
[InlineData("TenantB-UserB", "TenantB-Order1", HttpStatusCode.OK)]
[InlineData("TenantA-Admin", "TenantB-Order1", HttpStatusCode.Forbidden)]
public async Task CrossTenant_Access_ShouldBeBlocked(string userKey, string resKey, HttpStatusCode expected)
{
    var client = _factory.CreateClientAs(userKey);
    var resp = await client.GetAsync($"/api/orders/{_seed[resKey]}");
    resp.StatusCode.Should().Be(expected);
}
```

规则：
- 每个租户级实体都要有至少 1 组"跨租户拒绝"用例
- 每次新增/修改端点必须更新回归集
- CI 中标签 `@tenant-isolation` 的用例必须 100% 通过才可合入

## FLS 测试要点

```
- [ ] 列表接口：敏感字段已按策略脱敏
- [ ] 详情接口：同上
- [ ] 导出接口（Excel/CSV）：同上，且打开文件可见为脱敏文本
- [ ] 打印 / 预览 / PDF：同上
- [ ] Deny 策略字段：响应体中无该键（而非 null）
- [ ] 白名单用户访问：可见原值，且审计记录"敏感字段明文访问"
- [ ] 角色调整后缓存失效：下一请求立即生效
```

## 审计测试要点

```
- [ ] 每个写操作均落审计（成功 + 失败）
- [ ] 授权失败（403）单独审计（安全事件）
- [ ] CorrelationId 前后端 + 审计三方一致
- [ ] 审计表不允许 UPDATE/DELETE（用只写账户连接测试）
- [ ] 敏感字段在审计的 Before/After 中已脱敏
```

## 性能与一致性

```
- [ ] 权限缓存击穿场景：单 key 过期瞬间 1000 并发，授权结果一致
- [ ] 权限变更广播：集群内 < 2s 全量失效
- [ ] ABAC 判定 p99 < 5ms（含属性提供）
- [ ] 全局查询过滤器不引起 N+1
- [ ] 授权中间件 p99 < 2ms（本地缓存命中）
```

## 测试数据管理

- 每个租户独立的 seed 工厂：`SeedFactory.Tenant("A").WithUsers(...).WithRoles(...)`
- 用例间使用**独立 schema / 数据库快照**，避免串扰
- 敏感字段 seed 必须是"测试用占位值"（例如手机号 `138 0000 XXXX` 段）

## 测试评审清单

```
- [ ] 每个受保护端点完成权限矩阵用例
- [ ] 至少包含：无权限 / 跨租户 / FLS 三条用例
- [ ] 隔离回归集覆盖所有租户级实体
- [ ] 审计用例覆盖成功与失败两路径
- [ ] 性能用例包含缓存失效场景
- [ ] CI 中 @tenant-isolation 标签 100% 通过
- [ ] 新端点同步更新 OpenAPI，合约测试通过
- [ ] 前端 E2E 覆盖菜单可见性 + 按钮可见性 + 数据范围
```

## 常见测试反模式

- ❌ 仅测"正向用例"（有权限成功），不测"负向用例"（无权限拒绝）
- ❌ 用同一用户跑所有用例，无法验证跨租户
- ❌ 集成测试走 SQLite 内存库而生产是 SQL Server（全局过滤器行为差异）
- ❌ 共享测试数据库不清理，上一条用例污染下一条
- ❌ FLS 断言只校验 HTTP 200 而不校验响应体字段
- ❌ 授权性能测试用单用户 / 单权限码，不反映真实缓存场景
