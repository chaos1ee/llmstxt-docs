# @flow97/antd-schema

把后台页面描述成 Schema，而不是手写一整页 React 组件。这个包适合“页面结构重复、字段和动作经常变化、希望把页面定义数据化”的场景。

## 包里提供什么

- `SchemaRenderer`
  把 schema 渲染成页面
- 一组内置 renderer
  `viewport`、`card`、`flex`、`form`、`query-list`、`infinite-list`、`table`、`descriptions`、`service` 等
- 一组 schema 类型定义
  例如 `ViewportSchema`、`FormSchema`、`TableSchema`
- `renderArray`
  用于在自定义 renderer 中复用渲染管线

根入口导入时会自动注册内置 renderer，这是源码里的重要行为，所以推荐优先从根入口消费。

## 安装

```bash
pnpm add @flow97/antd-schema
```

常见依赖组合：

- `react` / `react-dom`
- `antd`
- `react-router`
- `@flow97/react-toolkit`

## 最小示例

```tsx
import { SchemaRenderer } from '@flow97/antd-schema'
import type { ViewportSchema } from '@flow97/antd-schema'

const schema: ViewportSchema = {
  type: 'viewport',
  body: [
    {
      type: 'card',
      title: '用户列表',
      body: [
        {
          type: 'query-list',
          identifier: 'users',
          rowKey: 'id',
          url: '/api/users',
          method: 'GET',
          columns: [
            { title: 'ID', dataIndex: 'id' },
            { title: '姓名', dataIndex: 'name' },
          ],
        },
      ],
    },
  ],
}

export default function UsersPage() {
  return <SchemaRenderer schema={schema} />
}
```

## 什么时候适合用

适合：

- 配置驱动型后台页面
- 后端或运营平台生成 JSON schema
- 想把字段、动作、布局沉淀为数据结构

不适合：

- 页面交互高度定制
- 组件状态机复杂
- 直接写 JSX 明显更简单的页面

## 文档导航

- [架构说明](./ARCHITECTURE.md)

## Schema 字段参考（由 `src/schemas` 定义）

下列说明与 **Zod schema 源码** 一致；`type` 为各节点必填字面量。除特别说明外，未列出的字段均为可选。

### 通用扩展（`baseSchema`）

多数节点在 `baseSchema` 上继续 `extend`：

| 字段 | 类型 | 说明 |
|------|------|------|
| `type` | `string` | 组件类型，对应注册表中的 renderer key。 |
| `storeId` | `string` | 可选。写入内部 store 的 key，用于表单值、HTTP 结果等（Service 侧注释称部分能力未支持时以渲染实现为准）。 |
| `storeIdRef` | `string` | 可选。引用其他节点的 `storeId`，从 store 读取关联数据。 |

`params` / `body` 等嵌套结构里，**仅当某个字符串值整段**为 `` `${...}` `` 形式时，花括号内会按 **JMESPath** 在当次请求使用的数据对象上求值并替换（例如列表筛选上下文、关联 `storeId` 的 store；具体 `source` 由各 renderer 传入）。普通字符串不会被当作表达式解析。

---

### `viewport`（`viewportSchema`）

| 字段 | 类型 | 说明 |
|------|------|------|
| `type` | `'viewport'` | 页面根，通常传给 `SchemaRenderer`。 |
| `body` | `any[]` | 可选。子 schema 数组。 |
| `data` | `Record<string, any>` | 可选。视窗级数据。 |

---

### `card`（`cardSchema`）

| 字段 | 类型 | 说明 |
|------|------|------|
| `type` | `'card'` | |
| `title` | `string` | 可选。卡片标题。 |
| `extra` | `any[]` | 可选。额外区域子节点。 |
| `body` | `any[]` | 可选。卡片主体子节点（lazy 递归）。 |

---

### `flex`（`flexSchema`）

| 字段 | 类型 | 说明 |
|------|------|------|
| `type` | `'flex'` | |
| `vertical` | `boolean` | 可选。主轴是否为纵向（`flex-direction: column`）。 |
| `wrap` | `boolean` | 可选。是否换行。 |
| `justify` | `'flex-start' \| 'center' \| 'flex-end' \| 'space-between' \| 'space-around' \| 'space-evenly'` | 可选。主轴对齐。 |
| `align` | `'flex-start' \| 'center' \| 'flex-end' \| 'baseline' \| 'stretch'` | 可选。交叉轴对齐。 |
| `gap` | `number \| string` | 可选。间距。 |
| `body` | `any[]` | 可选。子节点。 |
| `style` | `Record<string, any>` | 可选。内联样式对象。 |

---

### `div`（`divSchema`）

| 字段 | 类型 | 说明 |
|------|------|------|
| `type` | `'div'` | |
| `body` | `any[]` | 可选。子节点。 |
| `style` | `Record<string, any>` | 可选。样式。 |

---

### `span`（`spanSchema`）

| 字段 | 类型 | 说明 |
|------|------|------|
| `type` | `'span'` | 插入 `span`，用于文字或字符串模板。 |
| `text` | `string` | 文案（必填）。 |

---

### `link`（`linkSchema`）

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `type` | `'link'` | | |
| `href` | `string` | | 必填。链接地址。 |
| `text` | `string` | | 可选。显示文案。 |
| `target` | `'_blank' \| '_self' \| '_parent' \| '_top'` | `'_blank'` | |
| `params` | `Record<string, any>` | | 可选。跳转查询参数。 |

---

### `form`（`formSchema`）

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `type` | `'form'` | | |
| `name` | `string` | | 可选。表单名。 |
| `preserve` | `boolean` | | 可选。字段值保留策略。 |
| `fields` | `any[]` | | 可选。表单项 schema 数组（如 `input-text`、`select` 等）。 |
| `initialValues` | `Record<string, any>` | | 可选。初始值。 |
| `layout` | `'horizontal' \| 'inline' \| 'vertical'` | `'horizontal'` | 表单布局。 |
| `labelWidth` | `number \| string` | | 可选。标签宽度；数字表示 24 栅格列数，字符串走 flex。 |
| `labelAlign` | `'left' \| 'right'` | | 可选。标签对齐。 |
| `clearOnDestroy` | `boolean` | | 可选。卸载时是否清空。 |
| `gutter` | `number \| [number, number]` | `10` | 栅格间隔；元组为 `[横向, 纵向]`。 |

---

### 表单项公共字段（`formItemSchema`）

凡由 `formItemSchema` 扩展的字段控件均包含：

| 字段 | 类型 | 说明 |
|------|------|------|
| `label` | `string` | 标签（必填）。 |
| `name` | `string` | 字段名（必填）。 |
| `preserve` | `boolean` | 可选。 |
| `span` | `number` | 可选。栅格占位。 |
| `value` | `any` | 可选。默认值，对应 Form.Item `initialValue`。 |
| `rules` | `object[]` | 可选。校验规则数组，每项可含：`type`（`'string' \| 'number' \| 'boolean' \| 'integer'`）、`required`、`message`、`min`、`max`（`min`/`max` 语义依赖 `type`，见 schema 内 `.describe`）。 |

在此基础上各控件增加 `type` 字面量及专有字段：

#### `input-text`

| 字段 | 说明 |
|------|------|
| `type` | `'input-text'` |
| `clearable` | 可选 `boolean`。 |

#### `input-number`

| 字段 | 默认值 | 说明 |
|------|--------|------|
| `type` | `'input-number'` | |
| `clearable` | | 可选。 |
| `min` / `max` / `step` | | 可选 `number`。 |
| `width` | `'100px'` | 宽度。 |
| `controls` | `false` | 是否显示步进按钮。 |

#### `select`

| 字段 | 默认值 | 说明 |
|------|--------|------|
| `type` | `'select'` | |
| `allowClear` | | 可选。清除按钮。 |
| `virtual` | | 可选。虚拟滚动。 |
| `showSearch` | | 可选。可搜索。 |
| `width` | `'100px'` | |
| `options` | | 可选。`{ label: string; value: string }[]`。 |
| `optionFilterProp` | `'label'` | 搜索过滤字段。 |
| `source` | | 可选。数据源标识。 |

#### `date-picker`

| 字段 | 说明 |
|------|------|
| `type` | `'date-picker'` |
| `showTime` | 可选。含时间。 |
| `showNow` | 可选。快捷「此刻」。 |
| `showWeek` | 可选。展示周。 |
| `utc` | 可选。UTC。 |

#### `date-range`

| 字段 | 说明 |
|------|------|
| `type` | `'date-range'` |
| `allowEmpty` | 可选 `[boolean, boolean]`，两端是否允许为空。 |
| `showTime` / `utc` | 可选。 |

Schema 描述：**值为数组类型**，请求模板中常用 `range[0]`、`range[1]` 等形式映射。

---

### `table`（`tableSchema`）

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `type` | `'table'` | | |
| `rowKey` | `string` | | 可选。行唯一键字段。 |
| `columns` | `object[]` | | 可选。列：`title`（必填）、`dataIndex`（可选，支持点路径）、`width`、`align`、`tpl`（模板，schema 标注暂时不支持）。 |
| `footer` | `any[]` | | 可选。表格底部子节点。 |
| `api` | `string` | | 可选。GET 拉数地址，**优先级高于 `data`**。 |
| `data` | `any[] \| string` | | 可选。静态数据或变量引用。 |
| `bordered` | `boolean` | | 可选。 |
| `size` | `'small' \| 'middle' \| 'large'` | `'middle'` | |
| `tableLayout` | `'fixed' \| 'auto'` | | 可选。 |
| `pagination` | `false \| { current?, pageSize?, total? }` | `false` | |

说明：需要分页筛选、与表单联动请求时，优先使用 **`query-list`**。

---

### `query-list`（`queryListSchema`）

注释约定接口体：`{ rows: any[]; total: number }`（或兼容扁平 `rows` / `total` 的响应，以渲染器实现为准）。

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `type` | `'query-list'` | | |
| `identifier` | `string` | | 可选。实例唯一标识；未提供时由实现生成。 |
| `url` | `string` | | **必填**。请求地址。 |
| `code` | `string` | | 可选。权限编号。 |
| `method` | `'GET' \| 'POST'` | `'GET'` | |
| `onePage` | `boolean` | | 可选。 |
| `headers` | `Record<string, any>` | | 可选。请求头。 |
| `body` | `Record<string, any>` | | 可选。POST 体；支持模板占位。 |
| `params` | `Record<string, any>` | | 可选。查询参数；支持模板占位。 |
| `extra` | `any[]` | | 可选。表格额外配置。 |
| `actions` | `ButtonSchema[]` | | 可选。行操作按钮；行数据会沿树向下传递。 |
| `buttonsAlign` | `'left' \| 'right' \| 'bottom'` | | 可选。 |
| `formLayout` | 同 `form.layout` | | 与 `formSchema.shape.layout` 一致。 |
| `gutter` | 同 `form.gutter` | | |
| `fields` | 同 `form.fields` | | 筛选表单项。 |
| `initialValues` | 同 `form.initialValues` | | |
| `rowKey` | 同 `table.rowKey` | | |
| `tableLayout` | 同 `table.tableLayout` | | |
| `footer` | 同 `table.footer` | | |
| `columns` | 同 `table.columns` | | |

---

### `infinite-list`（`infiniteListSchema`）

注释约定接口体：`{ rows: any[]; rowKey: string; hasMore: boolean }`（以渲染器为准）。

| 字段 | 说明 |
|------|------|
| `type` | `'infinite-list'` |
| `identifier` | 可选，同 `query-list`。 |
| `url` | **必填**。 |
| `params` | 可选，支持模板占位。 |
| `code` | 可选，权限编号。 |
| `extra` / `actions` | 可选，含义同 `query-list`。 |
| `formLayout` / `gutter` / `fields` / `initialValues` | 同 `form` / `query-list` 对应字段。 |
| `rowKey` / `tableLayout` / `footer` / `columns` | 同 `table` 对应字段。 |

与 `query-list` 的差异（schema 层面）：**无** `method`、`headers`、`body`、`onePage`、`buttonsAlign` 字段定义。

---

### `descriptions`（`descriptionsSchema`）

接受的数据形态在源码注释中为：`{ label: string \| number; value: string \| number }[]`（具体取值以渲染器为准）。

| 字段 | 类型 | 说明 |
|------|------|------|
| `type` | `'descriptions'` | |
| `title` | `string` | 可选。 |
| `dataIndex` | `string` | 可选。JMESPath 表达式。 |
| `dataSource` | `any` | 可选。数据源。 |
| `column` | `number` | 可选。列数。 |

---

### `button`（`buttonSchema`）

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `type` | `'button'` | | |
| `code` | `string` | | 可选。权限编号。 |
| `buttonType` | `'primary' \| 'default' \| 'dashed' \| 'link' \| 'text'` | | 可选。 |
| `action` | `'modal' \| 'download'` | | 可选。 |
| `text` | `string` | | 可选。 |
| `data` | `any` | | 可选。自定义数据。 |
| `method` | `'GET' \| 'POST'` | | 可选，默认实现侧多为 GET。 |
| `url` | `string` | | 可选。请求地址。 |
| `params` / `body` | `Record<string, any>` | | 可选。 |
| `showLoading` | `boolean` | `true` | |
| `modal` | `{ title: string; body?: any[]; data?: Record<string, any>; width?: number }` | | `action === 'modal'` 时 **必填**。 |

**Zod 约束（`refine`）：**

- `action === 'download'` 时必须提供非空 `url`。
- `action === 'modal'` 时必须提供 `modal`。

---

### `service`（`serviceSchema`）

| 字段 | 类型 | 说明 |
|------|------|------|
| `type` | `'service'` | 请求外部服务，结果写入 store（与 `storeId` 配合使用）。 |
| `method` | `'GET' \| 'POST'` | 可选。 |
| `url` | `string` | 可选。 |
| `params` | `Record<string, any>` | 可选。 |
| `body` | `any` | 可选。 |

---

### 类型与导出入口

根包导出的 **Zod 对象与 TS 类型**包括：`baseSchema`、`viewportSchema`、`cardSchema`、`formSchema`、`inputTextSchema`、`inputNumberSchema`、`buttonSchema`、`serviceSchema`、`tableSchema` 及对应 `*Schema` 类型等（以 `lib/index.d.ts` 为准）。`query-list`、`infinite-list` 等部分节点仅有运行时 schema 文件，类型名未必从根入口导出，编写 JSON 时可对照本表与 `src/schemas/*.ts`。

## 内置 renderer 词汇表

根入口当前会自动注册这些 renderer：

- `button`
- `card`
- `date-picker`
- `date-range`
- `descriptions`
- `div`
- `flex`
- `form`
- `infinite-list`
- `input-number`
- `input-text`
- `link`
- `query-list`
- `select`
- `service`
- `span`
- `table`
- `viewport`

如果你的页面基本能被这些块表达出来，这个包就很适合。
