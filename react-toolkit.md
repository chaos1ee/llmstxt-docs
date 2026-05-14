# @flow97/react-toolkit

基于 React 19、Ant Design 6、React Router 7 与 TanStack Query 5 的中后台工具包。它把后台项目里反复出现的权限校验、菜单加载、应用壳、分页列表、无限滚动列表、弹层表单和页面保活整理成了一套可复用约定。

## 安装

```bash
pnpm add @flow97/react-toolkit
```

对等依赖由业务项目提供：

- `react` `^19.2.0`
- `react-dom` `^19.2.0`
- `antd` `^6.0.0`
- `react-router` `^7.9.6`
- `@tanstack/react-query` `^5.90.10`

全局样式只需要引入一次：

```ts
import '@flow97/react-toolkit/style.css'
```

## 快速开始

```tsx
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import { createRoot } from 'react-dom/client'
import { RouterProvider } from 'react-router'

import { AuthMode, CheckEndpoint, ToolkitProvider } from '@flow97/react-toolkit'
import '@flow97/react-toolkit/style.css'

import router from './router'

const queryClient = new QueryClient()

createRoot(document.getElementById('root')!).render(
  <QueryClientProvider client={queryClient}>
    <ToolkitProvider
      loginPath="/sign_in"
      homePath="/"
      sidebarCollapsible
      gameScoped
      authConfig={{
        mode: AuthMode.GROUP_BASED,
        checkEndpoint: CheckEndpoint.CHECK_V2,
      }}
    >
      <RouterProvider router={router} />
    </ToolkitProvider>
  </QueryClientProvider>,
)
```

使用前提：

- `ToolkitProvider` 必须放在 `QueryClientProvider` 里面
- `ToolkitProvider` 不支持嵌套
- `loginPath` 是必填项

## 导入约定

常用场景优先从根入口导入；只有当你需要根入口没有暴露的能力时，再走子路径。

```tsx
import { Layout, QueryList, ToolkitProvider, useFormModal } from '@flow97/react-toolkit'

import { createToolkitStore } from '@flow97/react-toolkit/stores'
import { PermissionRoutesWrapper } from '@flow97/react-toolkit/routes/permission'
import zhCN from '@flow97/react-toolkit/locale/zh_CN'
```

推荐约定：

- 根入口：页面最常用的组件、hooks、服务、常量
- `components/*`、`hooks/*`：想按模块管理 import 时使用
- `stores`、`routes/*`、`locale/*`：需要子路径专属导出时使用

## 模块导航

- [核心上下文与状态](#核心上下文与状态)
- [布局、权限与导航](#布局权限与导航)
- [列表与筛选](#列表与筛选)
- [弹层与可见性](#弹层与可见性)
- [页面保活](#页面保活)
- [基础展示组件](#基础展示组件)
- [国际化](#国际化)
- [页面与路由片段](#页面与路由片段)
- [类型与工具](#类型与工具)
- [子路径与导出映射](#子路径与导出映射)

## 核心上下文与状态

这一组 API 负责整个工具包的运行上下文、持久化状态和外部读写入口。

### `ToolkitProvider`

导入：

```tsx
import { ToolkitProvider } from '@flow97/react-toolkit'
```

作用：

- 向整个组件树注入 toolkit 全局 store
- 提供登录路径、首页路径、侧栏配置、游戏域配置、认证模式、文案包等上下文
- 所有依赖 `useToolkitStore`、`useAuth`、`useMenuList`、`useGames`、`Layout` 的能力都需要它

`ToolkitProviderProps` 实际是 `PropsWithChildren<DeepPartial<ContextSlice>>`。也就是说，除 `children` 外，你可以按需传入 `ContextSlice` 中的任意字段。

`ContextSlice` 字段如下：

| 字段                    | 类型                   | 默认值                                           | 说明                                                                             |
| ----------------------- | ---------------------- | ------------------------------------------------ | -------------------------------------------------------------------------------- |
| `useGameApiV2`          | `boolean`              | `false`                                          | 是否使用 `/api/game/list` 游戏接口；会影响 `useGames` 和 `GameSelect` 的取值逻辑 |
| `apiBaseUrl`            | `string \| undefined`  | `undefined`                                      | 请求 base URL，供内部 `ky` 客户端读取                                            |
| `loginPath`             | `string`               | `'/sign_in'`                                     | 登录页路径，缺失时会抛错                                                         |
| `homePath`              | `string`               | `'/'`                                            | 登录完成后默认首页                                                               |
| `sidebarWidth`          | `number \| undefined`  | `250`                                            | 侧边栏宽度                                                                       |
| `sidebarCollapsible`    | `boolean \| undefined` | `false`                                          | 侧边栏是否可折叠                                                                 |
| `gameScoped`            | `boolean \| undefined` | `false`                                          | 是否开启游戏作用域；为 `true` 时会出现游戏选择和 `RequireGame` 逻辑              |
| `locale`                | `Locale`               | `zh_CN`                                          | 文案包                                                                           |
| `authConfig`            | `AuthConfig`           | `{ mode: GAME_SCOPED, checkEndpoint: CHECK_V2 }` | 当前认证模式与权限校验接口版本                                                   |
| `userListSearchEnabled` | `boolean \| undefined` | `true`                                           | 内置权限用户页是否开启搜索等逻辑                                                 |

源码行为要点：

- `ToolkitProvider` 内部使用的是全局单例 store，而不是每次挂载都新建独立实例
- 外层已经存在 `ToolkitProvider` 时，内层会直接抛错
- `authConfig.mode` 不在 `AuthMode` 枚举内时会直接抛错

示例：

```tsx
<ToolkitProvider
  loginPath="/sign_in"
  homePath="/dashboard"
  sidebarCollapsible
  gameScoped
  authConfig={{
    mode: AuthMode.GROUP_BASED,
    checkEndpoint: CheckEndpoint.CHECK_V2,
  }}
>
  <App />
</ToolkitProvider>
```

### `useToolkitStore`

导入：

```tsx
import { useToolkitStore } from '@flow97/react-toolkit'
```

签名：

- `useToolkitStore(): ToolkitState`
- `useToolkitStore<T>(selector: (state: ToolkitState) => T): T`

用法：

- 不传 selector 时，返回完整状态
- 传 selector 时，返回选中的局部状态

源码行为要点：

- 这个 hook 依赖 `ToolkitProvider`
- 如果树上不存在 `ToolkitProvider`，会抛出带组件栈信息的错误

示例：

```tsx
const loginPath = useToolkitStore(state => state.context.loginPath)
const appId = useToolkitStore(state => state.game.appId)
const clear = useToolkitStore(state => state.clear)
```

### `toolkitStore`

导入：

```ts
import { toolkitStore } from '@flow97/react-toolkit'
```

作用：

- 暴露底层 `zustand` store，适合在非 React 场景中直接读写
- 根入口只导出这一项 store 能力

`ToolkitState` 结构：

| 字段      | 说明                        |
| --------- | --------------------------- |
| `context` | 当前上下文配置              |
| `token`   | token 与用户信息            |
| `game`    | 当前游戏 `appId`            |
| `layout`  | 侧栏折叠状态                |
| `nav`     | 当前展开/选中的导航项       |
| `clear()` | 清空 token 并清理持久化存储 |

示例：

```ts
toolkitStore.getState().token.setToken(jwt)
toolkitStore.getState().game.setAppId('1001')
toolkitStore.getState().clear()
```

### `createToolkitStore`

导入：

```ts
import { createToolkitStore } from '@flow97/react-toolkit/stores'
```

签名：

```ts
;(initProps?: Partial<ContextSlice>) => StoreApi<ToolkitState>
```

作用：

- 返回当前全局 `toolkitStore`
- 如果传入 `initProps`，会把它合并到现有 `context`

源码行为要点：

- 虽然名字叫 `createToolkitStore`，但它不是创建多实例 store 的工厂
- 它更新的是同一个全局单例 store
- 如果你需要隔离的 provider/store 实例，当前公开 API 不支持

示例：

```ts
createToolkitStore({
  loginPath: '/sign_in',
  homePath: '/dashboard',
})
```

### `stores` 子路径类型

导入：

```ts
import type {
  ContextSlice,
  GameSlice,
  LayoutSlice,
  NavSlice,
  TokenSlice,
  ToolkitState,
  UserInfo,
} from '@flow97/react-toolkit/stores'
```

说明：

- `UserInfo`：JWT 解码后的用户信息，当前包含 `authorityId`、`exp`
- `TokenSlice`：`token`、`user`、`setToken`、`clearToken`
- `GameSlice`：`appId`、`setAppId`
- `LayoutSlice`：`collapsed`、`toggleCollapsed`
- `NavSlice`：`openKeys`、`selectedKeys`、`setOpenKeys`、`setSelectedKeys`
- `ContextSlice`、`ToolkitState`：分别表示配置上下文和整棵 store 的完整类型

## 布局、权限与导航

这一组 API 负责页面壳、权限控制、游戏选择、导航菜单以及相关服务 hook。

### `Layout`

导入：

```tsx
import { Layout } from '@flow97/react-toolkit'
```

作用：

- 渲染后台常见的侧边栏 + 顶部栏 + 内容区结构
- 集成了 `UserDropdown`、游戏选择、导航权限过滤、`RequireGame`

`LayoutProps<T extends Game>`：

| 字段          | 类型                               | 说明             |
| ------------- | ---------------------------------- | ---------------- |
| `title`       | `string \| undefined`              | 侧栏顶部标题     |
| `titleStyle`  | `CSSProperties \| undefined`       | 标题样式         |
| `subtitle`    | `string \| undefined`              | 副标题           |
| `navigation`  | `NavigationConfig \| undefined`    | 导航项与加载状态 |
| `gameSelect`  | `GameSelectConfig<T> \| undefined` | 游戏选择器配置   |
| `headerExtra` | `HeaderExtraConfig \| undefined`   | 顶栏左右扩展区域 |

相关类型：

- `NavigationConfig`：`{ items?: NavItem[]; loading?: boolean }`
- `HeaderExtra`：`{ key: Key; children: ReactNode }`
- `HeaderExtraConfig`：`{ left?: HeaderExtra[]; right?: HeaderExtra[] }`
- `GameSelectConfig<T>`：`filter`、`options`、`routeModeRules`、`onChange`
- `Game`：至少包含 `game_id`，也允许带额外字段

源码行为要点：

- 侧栏宽度、是否可折叠、是否开启游戏域不在 `LayoutProps` 里，而是来自 `ToolkitProvider`
- `navigation.items` 会先做权限过滤，再渲染到 antd `Menu`
- `gameScoped` 为 `true` 时，顶栏会自动渲染 `GameSelect`
- 内容区会被 `RequireGame` 包一层；未选择游戏时不会继续渲染子内容

示例：

```tsx
<Layout
  title="Flow Admin"
  subtitle="Console"
  navigation={{ items: navItems, loading: isMenuLoading }}
  gameSelect={{
    routeModeRules: [{ rule: { path: '/overview', end: true }, mode: 'bypass' }],
    onChange: value => console.log('current game:', value),
  }}
>
  <KeepAliveOutlet />
</Layout>
```

### `GameSelectProps` 与路由模式类型

导入：

```ts
import type {
  GameSelectProps,
  GameSelectRouteMode,
  GameSelectRouteModeConfig,
  RouteMatchRule,
} from '@flow97/react-toolkit'
```

说明：

- `GameSelectProps<T>`：`filter`、`options`、`onChange`、`routeModeRules`
- `GameSelectRouteMode`：`'bypass' | 'locked' | 'default'`
- `GameSelectRouteModeConfig`：`{ rule: RouteMatchRule; mode: GameSelectRouteMode }`
- `RouteMatchRule`：`react-router` 的 `PathPattern` 或 `RegExp`

源码行为要点：

- `onChange` 实际拿到的是 `string`
- 当当前 `appId` 不在候选项里时，会自动选择第一个可用项
- 切换游戏时会先清空 `KeepAlive` 缓存，再失效除 `games.all` 之外的 React Query 查询

### `AuthButton`

导入：

```tsx
import { AuthButton } from '@flow97/react-toolkit'
```

作用：

- 根据权限码控制按钮的可用状态
- 内部基于 `useAuth` 自动发起权限请求

`AuthButtonProps` 继承 antd `ButtonProps`，并额外包含：

| 字段          | 类型                          | 说明                        |
| ------------- | ----------------------------- | --------------------------- | ---------- | ---------------------- |
| `code`        | `string                       | string[]                    | undefined` | 权限码；支持单个或多个 |
| `showLoading` | `boolean \| undefined`        | 请求中是否显示按钮 loading  |
| `config`      | `RequestOptions \| undefined` | 透传给 `useAuth` 的请求选项 |

源码行为要点：

- 权限不足时按钮会被禁用
- `showLoading` 为 `false` 时，加载中也会先禁用按钮
- 当前实现里，只要最终是 disabled 状态，就会显示“无权限” Tooltip；包括你手动传入 `disabled`

示例：

```tsx
<AuthButton code="user:create" type="primary">
  新建用户
</AuthButton>
```

### `RequireAuth`

导入：

```tsx
import { RequireAuth } from '@flow97/react-toolkit'
```

作用：

- 根据权限码决定是否渲染子树
- 无权限时展示 403 页面

`RequireAuthProps`：

| 字段           | 类型                          | 说明                           |
| -------------- | ----------------------------- | ------------------------------ |
| `code`         | `string`                      | 权限码，必填                   |
| `config`       | `RequestOptions \| undefined` | 请求选项                       |
| `redirectPath` | `string \| undefined`         | 403 页返回按钮目标，默认 `'/'` |
| `children`     | `ReactNode`                   | 有权限时渲染                   |

示例：

```tsx
<RequireAuth code="user:list" redirectPath="/dashboard">
  <UserListPage />
</RequireAuth>
```

### `RequireGame`

导入：

```tsx
import { RequireGame } from '@flow97/react-toolkit'
```

作用：

- 在开启游戏域时，阻止未选择游戏的页面继续渲染

`RequireGameProps`：

| 字段       | 类型                   | 说明                           |
| ---------- | ---------------------- | ------------------------------ |
| `bypass`   | `boolean \| undefined` | 是否跳过游戏校验，默认 `false` |
| `children` | `ReactNode`            | 子内容                         |

源码行为要点：

- 当 `gameScoped` 为 `false` 或 `bypass` 为 `true` 时，直接透传 children
- 当需要游戏域且正在拉取游戏列表时，会展示 `Spin`
- 当没有选中 `appId` 时，会展示空状态
- 当 `appId` 变化时，会用 `Fragment key={appId}` 强制重建子树

### `UserDropdown`

导入：

```tsx
import { UserDropdown } from '@flow97/react-toolkit'
```

作用：

- 渲染当前用户名称与退出登录菜单

参数：

- 无 props

源码行为要点：

- 展示的是 `user?.authorityId`
- 退出时会调用 `toolkitStore.clear()`，然后跳转到 `loginPath`

### `useAuth`

导入：

```tsx
import { useAuth } from '@flow97/react-toolkit'
```

签名：

```ts
useAuth(code?: string | string[], config?: RequestOptions)
```

作用：

- 调用权限校验接口
- 自动根据 `authConfig.checkEndpoint` 选择 `check` 或 `checkV2`

返回值：

- 保留了 React Query 的常用字段，如 `isLoading`、`error`、`refetch`
- `data` 的形状会根据参数变化：

| 调用方式                                  | `data` 类型               |
| ----------------------------------------- | ------------------------- |
| `useAuth('user:create')`                  | `boolean`                 |
| `useAuth(['user:create', 'user:update'])` | `Record<string, boolean>` |
| `useAuth()`                               | `true`，但不会发请求      |

源码行为要点：

- 当 `code` 为空时不会发请求
- 如果后端返回 `has_all`，单个或多个权限都会直接视为全部通过
- `config.isGlobalMode` 会参与 queryKey，避免不同作用域缓存串用

示例：

```tsx
const { data: canCreate, isLoading } = useAuth('user:create')
```

### `useMenuList`

导入：

```tsx
import { useMenuList } from '@flow97/react-toolkit'
```

作用：

- 拉取顶部/侧边导航菜单数据

返回值：

- React Query 返回对象
- `data` 为 `MenuListItem[]`

源码行为要点：

- 请求地址固定为 `/api/usystem/menu/navbar`
- `APP_ID_HEADER` 会根据当前 `appId` 自动写入
- 当前路径等于 `loginPath` 时不会发请求

### `useGames`

导入：

```tsx
import { useGames } from '@flow97/react-toolkit'
```

`UseGamesOptions`：

| 字段      | 类型                   | 说明         |
| --------- | ---------------------- | ------------ |
| `enabled` | `boolean \| undefined` | 是否启用请求 |

源码行为要点：

- `enabled` 默认等于 `gameScoped === true`
- `useGameApiV2` 为 `true` 时，请求 `/api/game/list`
- 否则请求 `/api/usystem/game/all`
- 请求头始终带 `App-ID: global`

### `useAuthConfig` / `useAuthMode` / `useIs*`

导入：

```tsx
import {
  useAuthConfig,
  useAuthMode,
  useIsDirectRole,
  useIsGameScoped,
  useIsGroupBased,
  useIsRoleWithGame,
} from '@flow97/react-toolkit'
```

返回值：

| Hook                  | 返回         |
| --------------------- | ------------ |
| `useAuthConfig()`     | `AuthConfig` |
| `useAuthMode()`       | `AuthMode`   |
| `useIsDirectRole()`   | `boolean`    |
| `useIsGameScoped()`   | `boolean`    |
| `useIsGroupBased()`   | `boolean`    |
| `useIsRoleWithGame()` | `boolean`    |

用途：

- 页面根据认证模式切换字段和行为
- 路由模块、权限页、角色页读取当前生效模式

### 常量：`SSO_URL`、`APP_ID_HEADER`、`FRONTEND_ROUTE_PREFIX`

导入：

```ts
import { APP_ID_HEADER, FRONTEND_ROUTE_PREFIX, SSO_URL } from '@flow97/react-toolkit'
```

说明：

| 常量                    | 值                                                                        | 用途                             |
| ----------------------- | ------------------------------------------------------------------------- | -------------------------------- |
| `SSO_URL`               | `https://idaas.ifunplus.cn/enduser/api/application/plugin_FunPlus/sso/v1` | 内置登录页拼接 SSO 登录/登出地址 |
| `APP_ID_HEADER`         | `'App-ID'`                                                                | 游戏作用域请求头                 |
| `FRONTEND_ROUTE_PREFIX` | `'/console/'`                                                             | 前端路由前缀常量                 |

### 枚举与配置：`CheckEndpoint`、`AuthMode`、`AuthConfig`、`WILDCARD`

导入：

```ts
import { AuthMode, CheckEndpoint, WILDCARD, type AuthConfig } from '@flow97/react-toolkit'
```

说明：

- `CheckEndpoint.CHECK`：对应 `/api/usystem/user/check`
- `CheckEndpoint.CHECK_V2`：对应 `/api/usystem/user/checkV2`
- `AuthMode`：
  - `DIRECT_ROLE`
  - `DIRECT_ROLE_WITH_GAME`
  - `GAME_SCOPED`
  - `GROUP_BASED`
- `AuthConfig`：`{ mode: AuthMode; checkEndpoint: CheckEndpoint }`
- `WILDCARD`：字符串 `'*'`

## 列表与筛选

这一组 API 用于构建“筛选表单 + 表格 / 无限加载列表”的后台标准页面。

### `FilterFormWrapper`

导入：

```tsx
import { FilterFormWrapper } from '@flow97/react-toolkit'
```

作用：

- 统一渲染筛选区容器、确认按钮、重置按钮和附加操作

`FilterFormWrapperProps`：

| 字段           | 类型                                  | 说明                        |
| -------------- | ------------------------------------- | --------------------------- |
| `children`     | `ReactNode`                           | 筛选区表单内容              |
| `onConfirm`    | `() => void \| Promise<void>`         | 查询按钮回调                |
| `onReset`      | `() => void`                          | 重置按钮回调                |
| `extras`       | `{ key: Key; children: ReactNode }[]` | 查询按钮旁的额外操作        |
| `isConfirming` | `boolean`                             | 查询按钮是否禁用            |
| `buttonsAlign` | `'left' \| 'right' \| 'bottom'`       | 按钮区域位置，默认 `'left'` |
| `showReset`    | `boolean`                             | 是否展示重置按钮            |

### `QueryList`

导入：

```tsx
import { QueryList, QueryListAction } from '@flow97/react-toolkit'
```

作用：

- 封装分页列表常见的查询、分页、错误处理和状态同步逻辑

`QueryListProps<Item, Values, Data>` 在 antd `TableProps` 的基础上，去掉了 `pagination`、`dataSource`、`loading`、`footer`，并额外增加：

| 字段              | 类型                                        | 说明                                   |
| ----------------- | ------------------------------------------- | -------------------------------------- |
| `identifier`      | `string \| undefined`                       | 实例 id；外部跨组件调用 store 时依赖它 |
| `code`            | `string \| undefined`                       | 权限码；无权限不发起列表请求           |
| `form`            | `FormInstance<Values> \| undefined`         | 外部表单实例                           |
| `refreshInterval` | `number \| undefined`                       | 轮询间隔，`0` 表示不轮询               |
| `onePage`         | `boolean \| undefined`                      | 是否隐藏分页                           |
| `defaultSize`     | `number \| undefined`                       | 默认每页条数，默认 `10`                |
| `pageSizeOptions` | `number[] \| undefined`                     | 每页条数选项                           |
| `request`         | `QueryListRequestConfigType<Values>`        | 请求配置，支持字符串、对象、函数       |
| `tableExtra`      | `ReactNode \| ((form, data?) => ReactNode)` | 表格额外区域                           |
| `renderForm`      | `(form) => ReactElement`                    | 自定义筛选表单                         |
| `afterSuccess`    | `(action, form, data?) => void`             | 请求成功回调                           |
| `afterError`      | `(error, action, form) => void`             | 请求失败回调                           |
| `dataAdapter`     | `QueryListDataAdapterConfig<Item, Data>`    | 将响应适配成 `items + total`           |
| `footer`          | `(data) => ReactNode`                       | 自定义表格 footer                      |
| `errorConfig`     | `QueryListErrorConfig<Data>`                | 自定义错误格式化与业务错误提取         |
| `buttonsAlign`    | `'left' \| 'right' \| 'bottom'`             | 透传给 `FilterFormWrapper`             |
| `showReset`       | `boolean`                                   | 透传给 `FilterFormWrapper`             |

相关类型：

- `QueryListAction`：`Confirm`、`Reset`、`Jump`、`Init`
- `QueryListPayload<Values>`：`{ page: number; size: number; filters?: Values }`
- `QueryListRequestConfig`：`url`、`method`、`body`、`searchParams`、`headers`、`cacheTime`、`staleTime`、`isGlobalMode`
- `QueryListErrorConfig<Data>`：
  - `formatError(error, data?)`
  - `getErrorFromSuccess(data)`
- `QueryListRef<Item, Values, Data>`：`data`、`dataSource`、`form`

源码行为要点：

- `request` 为字符串时，等价于只给了一个 `url`
- 默认数据格式假设为 `{ code, data: { list, total }, msg }`
- `errorConfig.getErrorFromSuccess` 可把“HTTP 成功但业务失败”转换成错误态
- `identifier` 不传时会自动生成

示例：

```tsx
<QueryList<User, { keyword?: string }>
  identifier="user-list"
  rowKey="id"
  request={({ page, size, filters }) => ({
    url: '/api/users',
    method: 'GET',
    searchParams: {
      page,
      size,
      keyword: filters?.keyword,
    },
  })}
  columns={[
    { title: 'ID', dataIndex: 'id' },
    { title: '姓名', dataIndex: 'name' },
  ]}
  renderForm={form => (
    <Form form={form} layout="inline">
      <Form.Item name="keyword" label="关键词">
        <Input allowClear />
      </Form.Item>
    </Form>
  )}
/>
```

### `useQueryListStore`

导入：

```tsx
import { useQueryListStore } from '@flow97/react-toolkit'
```

作用：

- 管理所有 `QueryList` 实例
- 支持在任意组件中按 `identifier` 主动刷新列表或同步 payload

hook 返回状态与方法：

| 字段 / 方法                                    | 说明                                |
| ---------------------------------------------- | ----------------------------------- |
| `instances`                                    | 当前已注册实例                      |
| `registerInstance(instance)`                   | 注册实例                            |
| `unregisterInstance(id)`                       | 注销实例                            |
| `updatePayload(id, payload, syncToComponent?)` | 更新 payload                        |
| `getPayload(id)`                               | 获取当前 payload                    |
| `refetch(id, payload?)`                        | 刷新实例；传 payload 时会先同步状态 |
| `getInstance(id)`                              | 获取实例                            |
| `getAllInstances()`                            | 获取全部实例                        |

静态方法：

- `useQueryListStore.registerInstance(id, url, queryKey, refetch, syncCallback?)`
- `useQueryListStore.getQueryKey(id)`

示例：

```tsx
const { refetch } = useQueryListStore()

await refetch('user-list', { page: 1 })
```

### `InfiniteList`

导入：

```tsx
import { InfiniteList } from '@flow97/react-toolkit'
```

作用：

- 封装无限滚动 / 加载更多列表

`InfiniteListProps<Item, Values, Data, PageParamType>`：

| 字段               | 类型                                                         | 说明                         |
| ------------------ | ------------------------------------------------------------ | ---------------------------- |
| `identifier`       | `string \| undefined`                                        | 实例 id                      |
| `code`             | `string \| undefined`                                        | 权限码                       |
| `form`             | `FormInstance<Values> \| undefined`                          | 外部表单实例                 |
| `refreshInterval`  | `number \| undefined`                                        | 自动刷新间隔                 |
| `request`          | `InfiniteListRequestConfigType<Values, PageParamType>`       | 请求配置                     |
| `tableExtra`       | `ReactNode \| ((form, data?) => ReactNode)`                  | 表格额外区域                 |
| `renderForm`       | `(form) => ReactElement`                                     | 自定义筛选表单               |
| `afterSuccess`     | `(form, data?) => void`                                      | 请求成功回调                 |
| `afterError`       | `(error, form) => void`                                      | 请求失败回调                 |
| `dataAdapter`      | `InfiniteListDataAdapter<Item, Values, Data, PageParamType>` | 自定义数据提取与分页推进逻辑 |
| `initialPageParam` | `PageParamType \| undefined`                                 | 初始分页参数，默认 `0`       |
| `footer`           | `(data) => ReactNode`                                        | 自定义 footer                |
| `buttonsAlign`     | `'left' \| 'right' \| 'bottom'`                              | 透传给 `FilterFormWrapper`   |
| `showReset`        | `boolean`                                                    | 透传给 `FilterFormWrapper`   |

相关类型：

- `PageParam`：`number | string | Record<string, any> | undefined`
- `InfiniteListPayload<Values>`：`{ page: number; filters?: Values }`
- `InfiniteListRequestConfig<Values, PageParamType>`：
  - `url`
  - `method`
  - `body`
  - `searchParams`
  - `headers`
  - `isGlobalMode`
  - `cacheTime`
  - `staleTime`
- `InfiniteListDataAdapter<Item, Values, Data, PageParamType>`：
  - `items(data, form)`
  - `hasMore(lastPage, allPages)`
  - `nextPageParam(lastPage, allPages, pageParam)`
- `InfiniteListRef<Item, Values, Data>`：
  - `data`
  - `dataSource`
  - `form`
  - `refetch`
  - `fetchNextPage`
  - `hasNextPage`
  - `isFetchingNextPage`

源码行为要点：

- 默认适配器假设每页结果里有 `data.list` 与 `hasMore` / `has_more`
- 如果 `initialPageParam` 是数字，默认 `nextPageParam` 会做 `+1`
- 权限码无权限时不会继续发请求

### `useInfiniteListStore`

导入：

```tsx
import { useInfiniteListStore } from '@flow97/react-toolkit'
```

作用：

- 与 `useQueryListStore` 类似，但额外支持跨组件调用 `fetchNextPage`

静态方法：

- `useInfiniteListStore.registerInstance(id, url, queryKey, refetch, fetchNextPage, syncCallback?)`
- `useInfiniteListStore.fetchNextPage(id)`
- `useInfiniteListStore.getQueryKey(id)`

示例：

```tsx
const { fetchNextPage, refetch } = useInfiniteListStore()

await refetch('feed-list')
await fetchNextPage('feed-list')
```

### `SelectAll`

导入：

```tsx
import { SelectAll } from '@flow97/react-toolkit'
```

作用：

- 为 antd 多选 Select 增加“全选 / 取消全选”能力

`SelectAllProps<V>` 在 `SelectProps` 基础上，额外强调这些字段：

| 字段                   | 类型                                    | 说明                                      |
| ---------------------- | --------------------------------------- | ----------------------------------------- |
| `options`              | `SelectProps<V>['options']`             | 必填，支持分组                            |
| `value`                | `V[] \| undefined`                      | 当前值，不包含“全选”占位值                |
| `onChange`             | `(value: V[], option: unknown) => void` | 回调拿到的永远是真实业务值                |
| `mode`                 | `'multiple' \| 'tags'`                  | 默认 `'multiple'`                         |
| `allLabel`             | `ReactNode \| undefined`                | “全选”展示文案                            |
| `allValue`             | `string \| undefined`                   | “全选”占位值，默认 `'_select_all_value_'` |
| `includeDisabledInAll` | `boolean \| undefined`                  | 全选时是否包含 disabled 项                |

源码行为要点：

- 第一次点“全选”会选中全部候选值
- 已全选时再次点“全选”会清空
- 下拉已选 tags 不会包含“全选”占位值

## 弹层与可见性

这一组 API 用于搭建 modal / drawer 及其表单版本，并复用统一的可见性 store。

### `useModal`

导入：

```tsx
import { useModal } from '@flow97/react-toolkit'
```

`UseModalProps` 在 antd `ModalProps` 的基础上，去掉了 `open`、`confirmLoading`、`onOk`、`onCancel`，并新增：

| 字段         | 类型                                                         | 说明           |
| ------------ | ------------------------------------------------------------ | -------------- |
| `content`    | `ReactNode \| ((operation: UseModalOperation) => ReactNode)` | 内容或渲染函数 |
| `onConfirm`  | `() => void \| Promise<void>`                                | 点击确认时执行 |
| `afterOpen`  | `() => void \| Promise<void>`                                | 打开后执行     |
| `afterClose` | `() => void \| Promise<void>`                                | 关闭后执行     |

`UseModalOperation`：

| 字段     | 说明           |
| -------- | -------------- |
| `hide()` | 手动关闭 modal |

返回值：

| 字段     | 说明                  |
| -------- | --------------------- |
| `id`     | 当前实例 id           |
| `show()` | 打开 modal            |
| `hide()` | 关闭 modal            |
| `modal`  | 要渲染到 JSX 里的节点 |

源码行为要点：

- `onConfirm` 成功后会自动关闭 modal
- 点击取消时也会执行 `afterClose`

示例：

```tsx
const { show, modal } = useModal({
  title: '删除确认',
  content: '确认删除这条记录？',
  onConfirm: async () => {
    await removeItem()
  },
})
```

### `useFormModal`

导入：

```tsx
import { useFormModal } from '@flow97/react-toolkit'
```

作用：

- 在 `useModal` 基础上加了一层 antd `Form`

`UseFormModalProps<Values, ExtraValues>`：

| 字段         | 类型                                                   | 说明                  |
| ------------ | ------------------------------------------------------ | --------------------- |
| `formProps`  | `Omit<FormProps, 'form'>`                              | 表单 props            |
| `form`       | `FormInstance<Values> \| undefined`                    | 外部表单实例          |
| `content`    | `ReactNode \| ((extraValues, operation) => ReactNode)` | 表单内容              |
| `onConfirm`  | `(values, extraValues) => void \| Promise<void>`       | 会先 `validateFields` |
| `onSuccess`  | `() => void`                                           | 提交成功回调          |
| `afterClose` | `(form) => void`                                       | 关闭后回调            |

`show(options?)`：

| 字段            | 类型                                    | 说明                               |
| --------------- | --------------------------------------- | ---------------------------------- |
| `initialValues` | `RecursivePartial<Values> \| undefined` | 打开时设置表单值                   |
| `extraValues`   | `ExtraValues \| undefined`              | 传给内容函数和提交函数的附加上下文 |

源码行为要点：

- 如果 `ExtraValues` 是具体类型而不是默认空对象类型，TS 会要求你在 `show()` 时传入 `extraValues`
- 提交成功后会自动关闭 modal

### `useModalStore`

导入：

```tsx
import { useModalStore } from '@flow97/react-toolkit'
```

返回：

- `VisibilityState`

说明：

- 这是 modal 专用的可见性 store
- 内部延迟初始化，避免模块初始化顺序问题

### `useDrawer`

导入：

```tsx
import { useDrawer } from '@flow97/react-toolkit'
```

`UseDrawerProps` 在 antd `DrawerProps` 基础上，去掉了 `open`、`confirmLoading`、`onClose`、`footer`，并新增：

| 字段                 | 类型                                                          | 说明                             |
| -------------------- | ------------------------------------------------------------- | -------------------------------- |
| `content`            | `ReactNode \| ((operation: UseDrawerOperation) => ReactNode)` | 内容                             |
| `onConfirm`          | `() => void \| Promise<void>`                                 | 确认回调                         |
| `afterOpen`          | `() => void \| Promise<void>`                                 | 打开后回调                       |
| `afterClose`         | `() => void \| Promise<void>`                                 | 关闭后回调                       |
| `footer`             | `ReactNode \| null`                                           | 自定义底部；传 `null` 可隐藏底部 |
| `confirmText`        | `string \| undefined`                                         | 确认按钮文案                     |
| `cancelText`         | `string \| undefined`                                         | 取消按钮文案                     |
| `confirmButtonProps` | `ComponentProps<typeof Button> \| undefined`                  | 确认按钮 props                   |
| `cancelButtonProps`  | `ComponentProps<typeof Button> \| undefined`                  | 取消按钮 props                   |

返回值：

| 字段             | 说明                      |
| ---------------- | ------------------------- |
| `id`             | 当前实例 id               |
| `show()`         | 打开抽屉                  |
| `hide()`         | 关闭抽屉                  |
| `confirmLoading` | 当前确认按钮 loading 状态 |
| `drawer`         | 要渲染的 drawer 节点      |

源码行为要点：

- `useDrawer` 和 `useModal` 的最大差异是：`onConfirm` 成功后不会自动关闭抽屉
- `afterClose` 只会在真正关闭时执行

### `useFormDrawer`

导入：

```tsx
import { useFormDrawer } from '@flow97/react-toolkit'
```

作用：

- 在 `useDrawer` 基础上集成 antd `Form`

参数结构与 `useFormModal` 基本一致，差异点只有返回值中的渲染节点变成了 `drawer`。

源码行为要点：

- 提交前会先 `validateFields`
- 提交成功后不会自动关闭抽屉
- 如果希望成功后关闭，建议自己在内容区调用 `operation.hide()`，或完全自定义 footer

### `useDrawerStore`

导入：

```tsx
import { useDrawerStore } from '@flow97/react-toolkit'
```

返回：

- `VisibilityState`

说明：

- 这是 drawer 专用可见性 store
- 同样是延迟初始化

### `createVisibilityStoreConfig`

导入：

```ts
import { createVisibilityStoreConfig } from '@flow97/react-toolkit'
```

签名：

```ts
;() => StateCreator<VisibilityState, [], [], VisibilityState>
```

作用：

- 返回一份通用的可见性 store 配置，适合自己基于 `zustand.create()` 复用

`VisibilityState` 字段：

| 字段                    | 说明                                           |
| ----------------------- | ---------------------------------------------- |
| `open`                  | `Map<number, boolean>`，记录每个 id 的打开状态 |
| `usedIds`               | `Set<number>`，记录占用中的 id                 |
| `isOpen(uuid)`          | 判断是否打开                                   |
| `show(uuid)`            | 打开指定实例                                   |
| `hide(uuid)`            | 关闭指定实例                                   |
| `hideAll()`             | 关闭全部                                       |
| `checkUniqueness(uuid)` | 判断 id 是否未被占用                           |
| `registerIds(uuid)`     | 注册 id                                        |
| `cleanup(uuid)`         | 清理 id 与状态                                 |

### `generateId`

导入：

```ts
import { generateId } from '@flow97/react-toolkit'
```

作用：

- 生成数值型唯一 id

源码行为要点：

- 由时间戳 + 随机数 + 自增计数组合而成
- modal / drawer 内部就使用它生成实例 id

示例：

```ts
const id = generateId()
```

## 页面保活

这一组 API 用于在 React Router 场景下缓存页面子树，减少列表页往返时的状态丢失。

### `KeepAliveProvider`

导入：

```tsx
import { KeepAliveProvider } from '@flow97/react-toolkit'
```

`KeepAliveProviderProps`：

| 字段             | 类型                   | 说明                   |
| ---------------- | ---------------------- | ---------------------- |
| `children`       | `ReactNode`            | 子组件                 |
| `maxCache`       | `number \| undefined`  | 最大缓存数，默认 `10`  |
| `clearOnUnmount` | `boolean \| undefined` | 卸载时是否清空全部缓存 |

作用：

- 提供保活缓存上下文
- 内部基于 LRU cache 管理缓存数量

### `KeepAlive`

导入：

```tsx
import { KeepAlive } from '@flow97/react-toolkit'
```

`KeepAliveProps`：

| 字段          | 类型                   | 说明                    |
| ------------- | ---------------------- | ----------------------- |
| `cacheKey`    | `string \| undefined`  | 缓存 key                |
| `children`    | `ReactElement`         | 要缓存的子树            |
| `disabled`    | `boolean \| undefined` | 是否禁用保活            |
| `getCacheKey` | `(location) => string` | 自定义缓存 key 生成逻辑 |

源码行为要点：

- 基于 React 19 `Activity` 实现
- 默认 key 为 `pathname + search`
- 组件切换 hidden / visible 时，state 会保留，但 effect 会重新执行

### `KeepAliveOutlet`

导入：

```tsx
import { KeepAliveOutlet } from '@flow97/react-toolkit'
```

`KeepAliveOutletProps`：

| 字段                 | 类型                   | 说明                          |
| -------------------- | ---------------------- | ----------------------------- |
| `disabled`           | `boolean \| undefined` | 禁用缓存                      |
| `cacheKeyPrefix`     | `string \| undefined`  | 自定义缓存 key 前缀           |
| `enableTransition`   | `boolean \| undefined` | 是否启用切换过渡，默认 `true` |
| `transitionDuration` | `number \| undefined`  | 过渡时长，默认 `200`          |

源码行为要点：

- 适合放在 `Layout` 内部，而不是根级 `Outlet`
- 未包 `KeepAliveProvider` 时会退化成普通 `Outlet`
- 会按路由层级自动为缓存 key 加前缀

### `useKeepAliveContext`

导入：

```tsx
import { useKeepAliveContext } from '@flow97/react-toolkit'
```

返回值：

- `cache`
- `addCache`
- `removeCache`
- `clearCache`
- `activateCache`
- `deactivateCache`
- `getCache`

注意：

- 只能在 `KeepAliveProvider` 内使用
- 否则会直接抛错

### `useKeepAlive`

导入：

```tsx
import { useKeepAlive } from '@flow97/react-toolkit'
```

返回值：

- `cache`
- `removeCache`
- `clearCache`
- `clearCacheByPath(path)`
- `clearCurrentCache()`
- `getCacheKeys()`
- `getAllCache()`
- `getCache()`
- `activateCache()`
- `deactivateCache()`

源码行为要点：

- 没有 `KeepAliveProvider` 时也不会报错，而是返回 no-op 版本
- `clearCacheByPath('/list')` 会清掉该路径及其查询参数变体

### `KeepAliveCacheItem`

导入：

```ts
import type { KeepAliveCacheItem } from '@flow97/react-toolkit'
```

结构：

| 字段      | 类型        | 说明         |
| --------- | ----------- | ------------ |
| `element` | `ReactNode` | 被缓存的节点 |
| `active`  | `boolean`   | 当前是否激活 |

更多示例见：

- [KeepAlive 组件说明](./src/components/keepAlive/README.md)

## 基础展示组件

这一组组件不依赖复杂状态，适合直接在业务页面中复用。

### `DynamicTags`

导入：

```tsx
import { DynamicTags } from '@flow97/react-toolkit'
```

`DynamicTagsProps`：

| 字段             | 类型                                       | 说明                     |
| ---------------- | ------------------------------------------ | ------------------------ |
| `initialTags`    | `string[] \| undefined`                    | 初始标签                 |
| `addable`        | `boolean \| undefined`                     | 是否允许新增             |
| `removable`      | `boolean \| undefined`                     | 是否允许删除             |
| `addCallback`    | `(addedTag: string) => Promise<boolean>`   | 返回 `true` 时才真正新增 |
| `removeCallback` | `(removedTag: string) => Promise<boolean>` | 返回 `true` 时才真正删除 |

源码行为要点：

- 内部支持双击标签进行编辑
- 编辑标签不会触发额外回调，只会更新本地状态
- `addCallback` / `removeCallback` 未传、返回 `false` 或 reject 时，都不会更新内部 tags

### `ExpandableParagraph`

导入：

```tsx
import { ExpandableParagraph } from '@flow97/react-toolkit'
```

`ExpandableParagraphProps`：

- 等价于 `Omit<ParagraphProps, 'ellipsis' | 'className'>`

源码行为要点：

- 固定两行省略
- 支持展开后再收起
- 自动带 `className="mb-0"`

### `Highlight`

导入：

```tsx
import { Highlight } from '@flow97/react-toolkit'
```

`HighlightProps`：

| 字段       | 类型          | 说明     |
| ---------- | ------------- | -------- | -------------------- |
| `texts`    | `Array<string | number>` | 需要高亮的关键词列表 |
| `children` | `ReactNode`   | 原始内容 |

源码行为要点：

- 内部会先把 children 渲染成 HTML 字符串，再替换匹配文本
- 最终通过 `dangerouslySetInnerHTML` 渲染
- 只适合展示可信内容，不适合直接处理用户输入的任意 HTML

### `Logo`

导入：

```tsx
import { Logo } from '@flow97/react-toolkit'
```

`LogoProps`：

- 等价于 `Omit<ImgHTMLAttributes<HTMLImageElement>, 'src' | 'alt'>`

说明：

- `src` 和 `alt` 已经内置
- 其他属性会透传给 `<img>`

## 国际化

### `useTranslation`

导入：

```tsx
import { useTranslation } from '@flow97/react-toolkit/locale'
```

返回值：

```ts
{
  t
}
```

其中：

```ts
t(key, data?)
```

用法：

- `key` 是 `Locale` 的嵌套路径字符串
- `data` 会用于 lodash `template` 插值

源码行为要点：

- 底层是 `lodash-es` 的 `get + has + template`
- key 不存在时会原样返回 key

示例：

```tsx
const { t } = useTranslation()

return <span>{t('global.back')}</span>
```

### `Locale`

导入：

```ts
import type { Locale } from '@flow97/react-toolkit/locale'
```

作用：

- 描述整个内置文案包的结构

当前内置命名空间包括：

- `global`
- `SignIn`
- `NotFound`
- `FilterFormWrapper`
- `FormModal`
- `GameSelect`
- `RequireGame`
- `UserDropdown`
- `User`
- `Role`
- `PermissionList`
- `RoleDetail`
- `InfiniteList`
- `Nav`

### `locale/*`

导入：

```ts
import zhCN from '@flow97/react-toolkit/locale/zh_CN'
import enGB from '@flow97/react-toolkit/locale/en_GB'
import jaJP from '@flow97/react-toolkit/locale/ja_JP'
import koKR from '@flow97/react-toolkit/locale/ko_KR'
```

说明：

- 这些子路径都是 default export
- 每个文件都导出一份完整 `Locale` 对象

示例：

```tsx
<ToolkitProvider loginPath="/sign_in" locale={enGB}>
  <App />
</ToolkitProvider>
```

## 页面与路由片段

### `SignIn`

导入：

```tsx
import { SignIn } from '@flow97/react-toolkit'
```

参数：

| 字段         | 类型                         | 说明               |
| ------------ | ---------------------------- | ------------------ |
| `extra`      | `ReactNode \| undefined`     | 页面右上角额外内容 |
| `title`      | `string \| undefined`        | Logo 旁标题        |
| `titleStyle` | `CSSProperties \| undefined` | 标题样式           |

源码行为要点：

- 已登录时会直接 `Navigate` 到 `homePath`
- 内部会根据当前 URL 拼接 SSO 登录回跳地址
- `?unregistered=true` 时会展示未注册提示

### `NotFound`

导入：

```tsx
import { NotFound } from '@flow97/react-toolkit'
```

参数：

| 字段          | 类型                  | 说明                             |
| ------------- | --------------------- | -------------------------------- |
| `redirectUrl` | `string \| undefined` | 存在时展示返回按钮并跳转到该地址 |

### `OperationLogList`

导入：

```tsx
import { OperationLogList } from '@flow97/react-toolkit'
```

参数：

- 无 props

说明：

- 这是一个基于 `QueryList` 封装好的操作日志页
- 内部已经定义好字段、筛选项和接口适配逻辑

### `menuRoutes`

导入：

```tsx
import { menuRoutes } from '@flow97/react-toolkit'
```

作用：

- 返回一段 React Router `<Route>` 片段
- 包含菜单管理模块的 `index`、`create`、`update/:id`

示例：

```tsx
<Route path="menu/*">{menuRoutes}</Route>
```

### `permissionRoutes`

导入：

```tsx
import { permissionRoutes } from '@flow97/react-toolkit'
```

作用：

- 返回权限管理模块的 `<Route>` 片段
- 包含 `user`、`role` 以及它们的详情页

示例：

```tsx
<Route path="permission/*">{permissionRoutes}</Route>
```

### `PermissionRoutesWrapper`

导入：

```tsx
import { PermissionRoutesWrapper } from '@flow97/react-toolkit/routes/permission'
```

说明：

- 根入口不导出它
- 它内部只是包了一层默认配置的 `ToolkitProvider` 和 `Outlet`

源码行为要点：

- 如果你需要自定义 `loginPath`、`locale`、`authConfig`，更建议自己写一层 wrapper，再把 `permissionRoutes` 塞进去

### `components/requirePermission`

导入：

```tsx
import { RequireAuth } from '@flow97/react-toolkit/components/requirePermission'
```

说明：

- 这个子路径当前实现上等价于 `components/requireAuth`
- 导出的仍然是 `RequireAuth` 和 `RequireAuthProps`
- 如果没有兼容历史 import 的需求，推荐统一使用 `requireAuth`

## 类型与工具

### 类型：`Permission`、`MenuListItem`、`RecursivePartial`、`JsonResponse`

导入：

```ts
import type { JsonResponse, MenuListItem, Permission, RecursivePartial } from '@flow97/react-toolkit'
```

说明：

| 类型                  | 结构 / 用途                                                         |
| --------------------- | ------------------------------------------------------------------- |
| `Permission`          | `{ label; value; route; method? }`，描述权限点                      |
| `MenuListItem`        | 菜单接口返回结构，包含 `permissions`、`front_route`、`order` 等字段 |
| `RecursivePartial<T>` | 深度可选类型，常用于 `show({ initialValues })`                      |
| `JsonResponse<T>`     | `{ code; message; data }` 通用接口包裹类型                          |

### `mixedStorage`

导入：

```ts
import { mixedStorage } from '@flow97/react-toolkit'
```

作用：

- 提供给 `zustand/middleware` 的 `StateStorage`

源码行为要点：

- 读取时优先 `sessionStorage`，再回退到 `localStorage`
- 写入和删除会同时操作两者

示例：

```ts
mixedStorage.setItem('debug-key', JSON.stringify({ value: 1 }))
```

## 子路径与导出映射

下面这张表按照 `package.json#exports` 汇总了公开入口。

| 子路径                                      | 主要内容                                                          |
| ------------------------------------------- | ----------------------------------------------------------------- |
| `@flow97/react-toolkit`                     | 常用组件、hooks、服务、常量、类型、`toolkitStore`、页面、路由片段 |
| `@flow97/react-toolkit/style.css`           | 全局样式                                                          |
| `@flow97/react-toolkit/constants`           | 常量与认证相关类型                                                |
| `@flow97/react-toolkit/services`            | `useAuth`、`useMenuList`、`useGames`                              |
| `@flow97/react-toolkit/types`               | 公共类型                                                          |
| `@flow97/react-toolkit/utils`               | `createVisibilityStoreConfig`、`generateId`、`mixedStorage`       |
| `@flow97/react-toolkit/stores`              | `toolkitStore`、`createToolkitStore` 及各 slice 类型              |
| `@flow97/react-toolkit/locale`              | `Locale`、`useTranslation`                                        |
| `@flow97/react-toolkit/locale/*`            | 各语言包 default export                                           |
| `@flow97/react-toolkit/components/*`        | 对应组件模块子路径                                                |
| `@flow97/react-toolkit/hooks/drawer`        | drawer 相关 hooks                                                 |
| `@flow97/react-toolkit/hooks/modal`         | modal 相关 hooks                                                  |
| `@flow97/react-toolkit/hooks/useAuthConfig` | 认证模式辅助 hooks                                                |
| `@flow97/react-toolkit/pages/*`             | 内置页面                                                          |
| `@flow97/react-toolkit/routes/*`            | 内置路由片段                                                      |

当前根入口公开导出如下：

- 组件：`AuthButton`、`DynamicTags`、`ExpandableParagraph`、`FilterFormWrapper`、`Highlight`、`KeepAlive`、`KeepAliveOutlet`、`KeepAliveProvider`、`Layout`、`Logo`、`RequireAuth`、`RequireGame`、`SelectAll`、`ToolkitProvider`、`UserDropdown`
- hooks：`useKeepAliveContext`、`useKeepAlive`、`useToolkitStore`、`useDrawer`、`useFormDrawer`、`useDrawerStore`、`useModal`、`useFormModal`、`useModalStore`、`useAuthConfig`、`useAuthMode`、`useIsDirectRole`、`useIsGameScoped`、`useIsGroupBased`、`useIsRoleWithGame`
- 列表：`QueryList`、`QueryListAction`、`useQueryListStore`、`InfiniteList`、`useInfiniteListStore`
- 服务：`useAuth`、`useMenuList`、`useGames`
- 常量 / 类型 / 工具：`SSO_URL`、`APP_ID_HEADER`、`FRONTEND_ROUTE_PREFIX`、`CheckEndpoint`、`AuthMode`、`AuthConfig`、`WILDCARD`、`Permission`、`MenuListItem`、`RecursivePartial`、`JsonResponse`、`createVisibilityStoreConfig`、`VisibilityState`、`generateId`、`mixedStorage`
- store / 页面 / 路由：`toolkitStore`、`SignIn`、`NotFound`、`OperationLogList`、`menuRoutes`、`permissionRoutes`

未公开导出的包内实现：

- `src/libs/ky.ts`
- `src/queryKeys.ts`
- 路由模块内部 services、pages、hooks、types 等

这些文件可以作为理解实现的参考，但不属于 package public API。
