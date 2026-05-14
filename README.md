# llmstxt-test

面向 LLM（如 Cursor / Claude / Codex 等）阅读的项目文档集合。仓库以 [llms.txt](https://llmstxt.org/) 的思路组织内容：把项目里面向人类的 README、设计文档、API 说明压缩成单文件、结构化、便于检索的 Markdown，让 AI 助手可以一次性吃下整个包的用法。

> 这是一个**文档仓库**，不包含运行时代码。它的产物是一组 `*.md` 文件，可以被 LLM 直接读取，也可以被进一步合成为 `llms.txt` / `llms-full.txt`。

## 仓库结构

```
.
├── antd-schema.md      # @flow97/antd-schema 的完整说明
├── react-toolkit.md    # @flow97/react-toolkit 的完整说明
└── README.md
```

每个 `*.md` 都是**一个 npm 包的全量文档**，包含：

- 包的定位与适用场景
- 安装与对等依赖
- 最小可运行示例
- 模块 / API / 字段参考
- 子路径与导出映射

## 已收录的包

### `antd-schema.md` — `@flow97/antd-schema`

把后台页面描述成 Schema、由 `SchemaRenderer` 统一渲染的库。文档涵盖：

- `SchemaRenderer` 与内置 renderer（`viewport` / `card` / `flex` / `form` / `query-list` / `infinite-list` / `table` / `descriptions` / `service` 等）
- 各 schema 节点字段参考（与 Zod 源码一致）
- 表单项控件（`input-text` / `input-number` / `select` / `date-picker` / `date-range`）字段定义
- JMESPath 模板占位规则

### `react-toolkit.md` — `@flow97/react-toolkit`

基于 React 19 + Ant Design 6 + React Router 7 + TanStack Query 5 的中后台工具包。文档涵盖：

- `ToolkitProvider` 与全局 store
- 布局、权限、导航（`Layout` / `RequireAuth` / `RequireGame` / `menuRoutes` / `permissionRoutes`）
- 列表与筛选（`QueryList` / `InfiniteList`）
- 弹层与可见性（`useFormModal` / `useFormDrawer`）
- 页面保活（`KeepAlive` / `KeepAliveOutlet`）
- 国际化、基础展示组件、工具函数
- `package.json#exports` 子路径映射

## 使用方式

### 给 LLM / AI 助手

直接把对应的 `*.md` 文件作为上下文喂给模型，例如：

- 在 Cursor 中用 `@antd-schema.md` 引用整篇文档
- 在 Codex / Claude 等 CLI 中通过 `--file` 或附件传入
- 拼接成 `llms-full.txt` 后部署到自家文档站，供爬虫消费

### 给人类读者

直接在 GitHub 上预览 Markdown 即可。每个文件本身就是一份可阅读的“单页文档”。

## 添加新包文档

1. 在仓库根目录新增 `<package-name>.md`
2. 文档建议保留以下章节，方便 LLM 解析：
   - 包定位 / 适用场景
   - 安装 + 对等依赖
   - 最小示例
   - 模块导航
   - API / 字段参考
   - 子路径与导出映射
3. 在本 README 的「已收录的包」一节追加条目

## 约定

- **以源码为准**：表格里的字段、默认值、行为说明来源于 Zod schema、TS 类型与运行时实现，避免凭印象描述。
- **保持紧凑**：能用一行表格说清楚的字段，不展开成段落；模型上下文是稀缺资源。
- **明确边界**：哪些是公开 API、哪些是内部实现，需要在文档里区分清楚。

## License

仓库内容用于内部文档协作，未单独声明许可证。文档中提及的第三方包遵循其各自的许可证。
