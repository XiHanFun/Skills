---
name: saas-rbac-abac-workflow:frontend
description: >-
  前端交付物：路由守卫、v-perm 指令、数据范围联动、FLS 展示约定与
  401/403 统一处理。用于 Vue 3 + TypeScript 权限前端实现。
---

# 前端开发交付物规范（Vue 3 + TypeScript）

前端**不做真授权**，只做"UI 可见性 + 体验"。所有数据与敏感判断最终以后端为准。

## 核心职责

1. 按后端下发的权限码驱动菜单 / 路由 / 按钮
2. 数据范围字段（部门/组织选择器）联动
3. FLS 展示态：区分"脱敏占位"与"无权查看"
4. 统一 403 / 401 处理与提示
5. 审计类交互（二次确认、原因输入）

## 权限元数据下发

登录后从后端拉取一次：

```ts
interface AuthMeta {
  tenantId: string;
  userId: string;
  isPlatform: boolean;
  permissions: string[];        // ["saas:user:read", ...]
  menus: MenuNode[];            // 已过滤后的菜单树
  dataScope: {
    kind: 'Self' | 'Dept' | 'DeptAndChild' | 'Tenant' | 'Custom' | 'Global';
    deptIds: string[];
  };
  version: string;              // 权限版本号，用于缓存失效
}
```

存储在 `useAuthStore`（Pinia），`version` 变化即整体刷新。

## 路由守卫

```ts
router.beforeEach(async (to) => {
  const auth = useAuthStore();
  if (to.meta.requiresAuth && !auth.isLoggedIn) return '/login';

  const needed = to.meta.permission as string | string[] | undefined;
  if (needed && !auth.hasAny(needed)) {
    return { name: '403' };
  }
});
```

路由定义：

```ts
{
  path: '/saas/users',
  component: () => import('@/views/saas/UserList.vue'),
  meta: { requiresAuth: true, permission: 'saas:user:read' },
}
```

## 按钮/元素级指令

```ts
// v-perm="'saas:user:create'"  或 v-perm="['a','b']"  或 v-perm="{any:['a','b']}"
app.directive('perm', {
  mounted(el, binding) {
    const auth = useAuthStore();
    const ok = matchPerm(binding.value, auth.permissions);
    if (!ok) el.parentNode?.removeChild(el);  // 删除而非隐藏，防样式残留
  },
});
```

模板：

```vue
<a-button v-perm="'saas:user:create'" @click="onCreate">新建用户</a-button>
```

## 组合式封装

```ts
export function usePerm() {
  const auth = useAuthStore();
  return {
    has:  (p: string) => auth.permissions.includes(p),
    any:  (ps: string[]) => ps.some(p => auth.permissions.includes(p)),
    all:  (ps: string[]) => ps.every(p => auth.permissions.includes(p)),
  };
}
```

## 数据范围联动

部门/组织选择器从 `auth.dataScope` 初始化：

```ts
const { dataScope } = useAuthStore();
const deptOptions = computed(() => {
  if (dataScope.kind === 'Global' || dataScope.kind === 'Tenant') return allDepts.value;
  return allDepts.value.filter(d => dataScope.deptIds.includes(d.id));
});
```

规则：
- 查询条件默认填充用户可见范围，禁止前端硬编码 `deptId=all`
- 导出接口复用列表查询的 DTO，**不单独暴露"全量导出"端点**

## FLS 展示约定

后端返回脱敏字符串时，前端需要做"可识别"的呈现：

| 后端返回 | 含义 | UI 呈现 |
|----------|------|---------|
| `"****"` | 全部脱敏 | 原样显示，tooltip："已脱敏" |
| `"138****5678"` | 部分脱敏 | 原样显示 |
| `"enc:..."` | 加密态 | 显示"●●●"，带"申请解密"按钮（走工单） |
| 字段缺失 | Deny 删除 | 列表列隐藏 / 详情区隐藏 |
| `null` | 业务空值 | 显示 `-` |

**禁止**：前端收到完整值后自行脱敏展示（说明后端 FLS 未生效，应修后端）。

## 请求层统一处理

```ts
// axios 拦截器
response.use(
  r => r,
  err => {
    if (err.response?.status === 401) { redirectLogin(); }
    if (err.response?.status === 403) {
      message.error('无权限执行此操作');
      logger.warn('forbidden', { url: err.config.url });
    }
    if (err.response?.status === 419) {  // 自定义：权限已变更
      useAuthStore().refresh();          // 拉新权限元数据
    }
    return Promise.reject(err);
  },
);
```

## 租户切换（平台超管 / 多租户用户）

- 切换时**清空**所有路由缓存、store、组件 keep-alive
- 刷新权限元数据后再放行路由
- URL 中**不**携带 `tenantId`（避免被分享/截取）

## 前端评审清单

```
- [ ] 菜单来自后端下发，而非前端静态拼装
- [ ] 所有敏感按钮 v-perm 或 usePerm 判定
- [ ] 路由 meta.permission 与后端权限码一致
- [ ] 列表查询默认填充数据范围
- [ ] 无自行脱敏逻辑（依赖后端 FLS）
- [ ] 401/403/419 有统一处理
- [ ] 登出/切租户时清空 store 与缓存
- [ ] 权限错误可追溯（含 correlationId）
```

## 常见前端反模式

- ❌ 用 `v-if="userInfo.roleName === 'admin'"` 判定业务权限
- ❌ 在前端路由表里硬编码"谁能看什么菜单"
- ❌ 前端过滤敏感字段（"后端返回了，前端藏起来"）
- ❌ 用 localStorage 长期持久化权限列表而不校验 `version`
- ❌ 导出接口单独走"管理员专用 API"，绕过列表 DTO 的 FLS
- ❌ 在 URL / Query / Hash 携带 `tenantId` 参与后端鉴权
