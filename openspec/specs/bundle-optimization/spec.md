# bundle-optimization Specification

## Purpose

定义 `@eternalheart/react-file-preview` 与 `@eternalheart/vue-file-preview` 两个发布包的按需加载、依赖外部化、子路径入口、CSS 拆分与 CJS 兼容等约束，以保证使用者获得按需、零 BREAKING 的安装与运行体验。
## Requirements
### Requirement: Renderer 按需动态加载

`FilePreviewContent` 渲染入口在解析出 `fileType` 之后，SHALL 通过动态 `import()` 加载对应 renderer 模块；MUST NOT 在主入口模块顶部静态 import 任何 renderer 实现。

#### Scenario: 打开 PDF 时按需加载

- **WHEN** 用户首次在 `FilePreviewModal` 中打开 PDF 文件
- **THEN** 浏览器发起对 `chunks/pdf-*.mjs` 的网络请求
- **AND** 在打开图片之前，浏览器 MUST NOT 加载 PDF chunk

#### Scenario: 仅渲染图片时不下载 PDF chunk

- **WHEN** 用户只渲染图片类型文件
- **THEN** 浏览器的 Network 面板中 MUST NOT 出现包含 `pdfjs-dist`、`react-pdf`、`pptx-preview`、`docx-preview` 的 chunk 请求

#### Scenario: Renderer 加载中显示占位

- **WHEN** 异步 chunk 正在加载且尚未完成
- **THEN** UI 中 SHALL 显示统一的加载占位（带 i18n 文本）
- **AND** 占位组件本身 MUST 位于主入口 chunk 中

### Requirement: 重型依赖外部化

构建配置 SHALL 按产物格式区分"外部化"与"按需 chunk 内联"两类依赖。

**ESM 产物(`lib/index.mjs` 及其 lazy chunk)** 的 rollup `external` 列表:

- **React 包**:`react`、`react-dom`、`react/jsx-runtime`、`react-pdf`、`react-markdown`、`framer-motion`、`lucide-react`、`mammoth`、`docx-preview`、`pptx-preview`、`exceljs`(含子路径)、`foliate-js`(含子路径)、`@likecoin/epub-ts`、`jszip`、`remark-gfm`、`remark-math`、`rehype-katex`、`rehype-raw`、`katex`(含子路径)、`shiki`(含子路径)、`video.js`、`heic2any`、`@jsquash/avif`、`utif`、`ag-psd`
- **Vue 包**:`vue`、`lucide-vue-next`、`markdown-it`、`@traptitech/markdown-it-katex`、`katex`(含子路径)、`shiki`(含子路径)、`mammoth`、`pptx-preview`、`exceljs`(含子路径)、`foliate-js`(含子路径)、`@likecoin/epub-ts`、`jszip`、`video.js`、`heic2any`、`@jsquash/avif`、`utif`、`ag-psd`

ESM 产物 MUST NOT 把以下依赖声明为 external,它们 SHALL 跟随对应 renderer 的动态 `import()` 边界被打入 lazy chunk:

- `@kenjiuno/msgreader`(由 Msg renderer 主线程引用 → 落入 Msg renderer chunk,约 323 KB / gzip 187 KB)
- `opentype.js`(由 Font renderer 主线程引用 → 落入 Font renderer chunk)
- `pdfjs-dist`(由 PDF renderer 引用 → 落入 PDF renderer 与 worker chunk)

以下依赖 SHALL NOT 出现在 external 列表中，且仅作为构建依赖保留在 `devDependencies`：

- `x-data-spreadsheet`、`jsonc-parser`（React/Vue 的 ESM 与 CJS 均内联）
- `jpeg2000`
- `@cornerstonejs/codec-openjpeg`（两者由 Core 通过 `?worker&inline` 内联到 JP2 worker）

**CJS 产物(`lib/index.cjs`)** SHALL 维持与 ESM 一致的 external 列表，**额外补充** `@kenjiuno/msgreader`、`opentype.js`、`pdfjs-dist` 继续 external，以避免 `inlineDynamicImports: true` 下单文件体积进一步膨胀。

外部化的依赖 SHALL 继续保留在 `package.json#dependencies` 字段(除 `react` / `react-dom` / `vue` 保留在 `peerDependencies`),以保证使用者 `npm install` 时自动拉取,无需任何手动安装步骤。仅 ESM 内联、CJS external 的依赖同样 SHALL 保留在 `dependencies` 字段。ESM/CJS 均内联的依赖 MUST NOT 保留在 `dependencies`，避免消费者重复安装。

`@eternalheart/file-preview-core` MUST 继续内联打包(因其未发布到 npm)。

#### Scenario: ESM 外部化依赖不出现在产物中

- **WHEN** 在 `lib/index.mjs` 与 `lib/chunks/*.mjs` 中 grep `from "pdfjs-dist"`、`from "shiki"`、`from "katex"` 等模式
- **THEN** 应能匹配到 `import` 语句,而 MUST NOT 出现这些模块的实现代码

#### Scenario: ESM chunk 内联的依赖实现代码出现在对应 chunk

- **WHEN** 在 `lib/index.mjs` 与 `lib/chunks/*.mjs` 中 grep `from "@kenjiuno/msgreader"`、`from "opentype.js"`
- **THEN** MUST NOT 匹配到这些模块作为外部 import 出现
- **AND** 对应的实现代码 SHALL 出现在 Msg renderer 或 Font renderer 的 chunk 中

#### Scenario: jpeg2000 / openjpeg 完全无产物引用

- **WHEN** 在 `lib/**/*.mjs` 与 `lib/index.cjs` 中 grep `jpeg2000` 与 `@cornerstonejs/codec-openjpeg`
- **THEN** MUST NOT 匹配到任何 import / require 语句(仅 `jp2Loader-*.mjs` 的 base64 内部可能含其代码,这是 worker base64 字符串,不是模块引用)

#### Scenario: CJS 产物继续外部化 msgreader 与 opentype.js

- **WHEN** 在 `lib/index.cjs` 中 grep `require("@kenjiuno/msgreader")`、`require("opentype.js")`
- **THEN** 应能匹配到 `require` 语句,而 MUST NOT 出现这些模块的实现代码

#### Scenario: 运行时依赖与构建依赖分离

- **WHEN** 检查 `packages/react-file-preview/package.json` 与 `packages/vue-file-preview/package.json`
- **THEN** 外部化依赖及仅 ESM 内联的依赖 SHALL 出现在 `dependencies` 字段中
- **AND** `x-data-spreadsheet`、`jsonc-parser` SHALL 仅出现在 `devDependencies` 中
- **AND** `jpeg2000`、`@cornerstonejs/codec-openjpeg` SHALL 仅出现在 Core 的 `devDependencies` 中
- **AND** 使用者执行 `npm install @eternalheart/react-file-preview`(或对应 vue 包)后,其 `node_modules` 中自动包含全部依赖

#### Scenario: core 包仍内联

- **WHEN** 检查 `lib/index.mjs`
- **THEN** `@eternalheart/file-preview-core` 的代码 SHALL 内联其中
- **AND** `lib/index.mjs` MUST NOT 出现 `from "@eternalheart/file-preview-core"` 的 import 语句

### Requirement: 升级零 BREAKING

使用者从前一版本（1.3.x）升级到本次发布版本（1.4.0）时，SHALL 不需要修改任何业务代码、不需要手动安装任何新的 npm 包。主入口的所有现有导出（`FilePreviewModal`、`FilePreviewEmbed`、`FilePreviewContent`、`normalizeFile`、`normalizeFiles`、`configurePdfjs`、`pdfjs`、`VERSION`、`LocaleProvider`、`useTranslator`、`useLocale` 等）SHALL 保持完全一致的签名与运行时行为。

#### Scenario: 旧代码无需改动

- **WHEN** 使用者把 `@eternalheart/react-file-preview` 从 `1.3.x` 升级到 `1.4.0`
- **AND** 仅执行 `pnpm install` 而不修改任何源码
- **THEN** 应用应能正常构建与运行，全部 18 种文件类型仍可正常渲染

#### Scenario: 公共 API 兼容

- **WHEN** 对比 `1.3.x` 与 `1.4.0` 的 `lib/index.d.ts`
- **THEN** 所有公共导出名与类型 SHALL 一致或为兼容超集（不允许移除、不允许收窄类型）

### Requirement: React 与 Vue 包构建策略一致

两个包的 vite 配置 SHALL 采用一致的构建策略：相同的 chunk 命名规则、相同的 external 治理标准、相同的 CSS 拆分规则、相同的 `peerDependencies` / `optionalDependencies` 治理思路。差异仅限于框架专用依赖（react vs vue）。

#### Scenario: 构建配置对齐审查

- **WHEN** 对比 `packages/react-file-preview/vite.config.ts` 与 `packages/vue-file-preview/vite.config.ts`
- **THEN** 二者 `cssCodeSplit`、`inlineDynamicImports`、entry 拆分逻辑、chunk 命名规则、external 模式 SHALL 一致或镜像对应

### Requirement: 发布产物不包含 SourceMap

React、Vue 与 Core 的 npm 发布产物 MUST NOT 生成 JavaScript source map 或声明文件 source map。

#### Scenario: 发布目录无 SourceMap

- **WHEN** 完成三个包的构建
- **THEN** `packages/*/lib` 中 MUST NOT 存在 `.map` 文件

### Requirement: CJS 兼容保留

为兼容旧 Node 工具链与 Jest，每个包 SHALL 继续产出 `lib/index.cjs`。允许 CJS 产物因不支持顶层 await 而采用单文件内联（`inlineDynamicImports: true`），但 ESM 产物 MUST 保持代码分割。`package.json#exports` SHALL 同时声明 import / require 两条路径。

#### Scenario: CJS 仍可导入

- **WHEN** Node CommonJS 工程执行 `const { FilePreviewModal } = require('@eternalheart/react-file-preview')`
- **THEN** 能正确获得组件（不要求体积达 ESM 标准）

#### Scenario: ESM 保持分割

- **WHEN** 现代打包器以 ESM 解析包
- **THEN** 主入口 `lib/index.mjs` SHALL 是分割后的壳，而非单文件内联

### Requirement: 使用者零额外配置即可使用全部 renderer
使用者执行 `npm install @eternalheart/react-file-preview`（或 Vue 对应包）后 SHALL 能使用全部 renderer，无需安装框架包 `dependencies` 已声明的额外依赖；WOFF2 SHALL 使用浏览器原生 FontFace 路径，不依赖 `wawoff2`。

#### Scenario: pnpm 严格模式下零配置使用 Font renderer
- **WHEN** 使用者在 pnpm 严格模式项目中预览 WOFF2 字体
- **THEN** 浏览器 MUST 通过原生 FontFace 完成预览
- **AND** 用户项目 MUST NOT 安装或配置 `wawoff2`

#### Scenario: pnpm 严格模式下零配置使用 Msg renderer
- **WHEN** 使用者在 pnpm 严格模式项目中预览 MSG 文件
- **THEN** Msg renderer MUST 能解析并展示文件
- **AND** 用户项目 MUST NOT 手动配置 `stream` 或 `@kenjiuno/msgreader`

#### Scenario: monorepo 示例不需要依赖别名
- **WHEN** 检查 React 与 Vue 示例的 Vite 配置
- **THEN** `resolve.alias` MUST NOT 包含 renderer 依赖的补丁别名

### Requirement: 库构建时消除 stream 模块引用

`@eternalheart/react-file-preview` 与 `@eternalheart/vue-file-preview` 的 `vite.config.ts` SHALL 在 `resolve.alias` 中把 Node 内置模块 `stream` 重定向到库内自带的空 stub(`packages/<pkg>/src/shims/stream-stub.ts`)。

stub 文件 SHALL 至少导出 `Transform`、`Readable`、`Writable`、`Duplex`、`PassThrough` 五个具名导出与一个默认导出,且全部为 `undefined` 或空对象,以满足 `iconv-lite` 的 `stream_module.Transform` 访问形式。

#### Scenario: 库产物中无 stream 引用残留

- **WHEN** 在 `lib/index.mjs` 与 `lib/chunks/*.mjs` 中 grep `from ['"]stream['"]` 与 `require\(['"]stream['"]\)`
- **THEN** MUST NOT 匹配到任何一条结果

#### Scenario: 构建过程无 stream externalized 警告

- **WHEN** 执行 `pnpm --filter @eternalheart/react-file-preview build`(或 vue 对应包)
- **THEN** 构建输出 MUST NOT 包含 "Module 'stream' has been externalized for browser compatibility" 字样的警告
