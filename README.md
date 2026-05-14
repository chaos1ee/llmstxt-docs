# llmstxt-docs

> 面向 LLM（Cursor / Claude / Codex 等）阅读的项目文档集合。
> 内容遵循 [llms.txt](https://llmstxt.org/) 思路：单文件、结构化 Markdown，便于模型一次性吃下整个包的用法。

- **在线索引**：<https://chaos1ee.github.io/llmstxt-docs/>
- **仓库形态**：纯文档仓库，不含运行时代码；产物是一组 `*.md`，可被 LLM 直接读取，也可合成 `llms.txt` / `llms-full.txt`。

## 快速使用

### 在 LLM 里引用

| 场景 | 推荐方式 |
| --- | --- |
| Cursor | 用 `@antd-schema.md`、`@react-toolkit.md` 引用整篇 |
| Claude / Codex CLI | 通过附件或 `--file` 传入对应 `*.md` |
| 在线对话 | 把站点链接 <https://chaos1ee.github.io/llmstxt-docs/> 发给模型，让其按页面打开链接 |
| 文档站 / 爬虫 | 拼接为 `llms-full.txt` 后部署 |

### 在本地预览

```bash
npx serve .
# 访问 http://localhost:3000/ ，点开各 .md 文件
```

直接在 GitHub 网页上预览 Markdown 同样可读。

## 已收录的包

| 文件 | 包 | 摘要 |
| --- | --- | --- |
| [`antd-schema.md`](./antd-schema.md) | `@flow97/antd-schema` | Schema 驱动后台页：`SchemaRenderer`、内置 renderer、Zod 字段参考、JMESPath 模板。 |
| [`react-toolkit.md`](./react-toolkit.md) | `@flow97/react-toolkit` | React 19 + AntD 6 中后台工具包：`ToolkitProvider`、布局权限、列表、弹层、保活、exports 映射。 |

每个 `*.md` 都是**一个 npm 包的全量文档**，固定包含：

- 包定位与适用场景
- 安装与对等依赖
- 最小可运行示例
- 模块 / API / 字段参考
- 子路径与导出映射

## 仓库结构

```
.
├── README.md
├── index.html          # GitHub Pages 索引页
├── antd-schema.md      # @flow97/antd-schema 的完整说明
└── react-toolkit.md    # @flow97/react-toolkit 的完整说明
```

## 添加新包文档

1. 在仓库根目录新增 `<package-name>.md`。
2. 建议保留以下章节，便于 LLM 解析：
   - 包定位 / 适用场景
   - 安装 + 对等依赖
   - 最小示例
   - 模块导航
   - API / 字段参考
   - 子路径与导出映射
3. 在 [`README.md`](./README.md) 的「已收录的包」表格与 [`index.html`](./index.html) 的卡片列表里追加条目。

## 编写约定

- **以源码为准**：字段、默认值、行为说明来源于 Zod schema、TS 类型与运行时实现，不凭印象。
- **保持紧凑**：能用一行表格说清楚的字段，不展开成段落；模型上下文是稀缺资源。
- **明确边界**：区分公开 API 与内部实现。

## License

仓库内容用于内部文档协作，未单独声明许可证。文档中提及的第三方包遵循其各自的许可证。
