# AGENTS.md · 项目说明

> 本文件面向在本仓库里工作的 AI 代理与人类协作者。开工前先读这里，再看 `PRD.md`（做什么）和 `TECH_DESIGN.md`（怎么做）。

---

## 1. 项目是什么

一个**纯静态的个人网站**，包含两个页面：

| 页面 | 路径 | 面向 | 能力 |
| --- | --- | --- | --- |
| 前台 | `index.html` | 访客 | 只读展示个人简介（Hero + 关于 + 技能）与项目卡片 |
| 后台 | `admin.html` | 站主本人 | 本地口令进入，编辑个人信息、增删改排序项目、导入导出备份 |

**核心特征**：没有后端、没有数据库、没有构建步骤。数据存在浏览器 `localStorage` 里，因此"后台"是纯前端的管理界面，不是真的服务端后台。

**一句话**：`HTML + CSS + 原生 JS`，打开浏览器就能跑，内容自己改。

---

## 2. 技术栈与硬约束

### 2.1 允许

- HTML5、CSS3（含 CSS 变量、Grid、Flexbox）
- 原生 JavaScript（ES2020 语法：可选链、`??`、`async/await`、模板字符串）
- 浏览器原生 API：`localStorage`、`sessionStorage`、`Canvas`、`FileReader`、`Blob`

### 2.2 禁止

- ❌ 任何框架 / 库（React、Vue、jQuery、Tailwind、Bootstrap……）
- ❌ 任何构建工具（Vite、Webpack、Rollup、SCSS、TypeScript 编译）
- ❌ 任何后端服务、数据库、云函数
- ❌ 任何外部 CDN、网络字体、统计脚本、第三方图标库
- ❌ `type="module"` 脚本（**原因见第 6.1 节**）
- ❌ `npm install` 引入运行时依赖（仅允许可选的本地静态服务器，如 `npx serve`）

---

## 3. 如何本地运行

### 3.1 方式 A：本地静态服务器（推荐）

任选其一，在项目根目录执行：

```bash
# 有 Node.js
npx serve .

# 有 Python 3
python -m http.server 8000

# 有 PHP
php -S localhost:8000
```

然后访问：

- 前台：<http://localhost:8000/index.html>
- 后台：<http://localhost:8000/admin.html>

### 3.2 方式 B：直接双击 `index.html`

可以打开，但有两个限制：

1. `file://` 不是安全上下文，`crypto.subtle`、`createImageBitmap` 等能力可能不可用（头像压缩会受影响）。
2. **`file://` 与 `http://localhost:8000` 的 `localStorage` 是两套独立数据**，在一种方式下录入的内容，另一种方式看不见。

### 3.3 ⚠️ 数据隔离须知

`localStorage` 按 **协议 + 域名 + 端口** 隔离：

| 访问方式 | 数据空间 |
| --- | --- |
| `file:///.../index.html` | 一套 |
| `http://localhost:8000` | 一套 |
| `https://yourname.github.io` | 一套 |

**三个入口不要来回切**。日常开发固定用一个地址（建议 `http://localhost:8000`），上线后固定用线上地址，迁移靠后台的"导出 / 导入 JSON"。

### 3.4 自检

1. 打开 `index.html`，应看到默认姓名"你的名字"和 3 张示例项目卡。
2. 打开 `admin.html`，应只看到口令输入框。
3. 输入口令进入，改姓名保存，回前台刷新，姓名已变。

默认口令见 `js/auth.js` 的 `PASSWORD` 常量（当前为 `123456`，仅用于防误触，无实际安全性）。

---

## 4. 目录结构要求

```text
my-personal-website/
├── index.html              # 前台首页（唯一入口，只读）
├── admin.html              # 后台管理页（唯一入口，读写）
├── css/
│   ├── base.css            # CSS 变量、reset、排版、按钮/输入框/toast 等通用组件
│   ├── site.css            # 前台样式：hero / about / cards / footer
│   └── admin.css           # 后台样式：双栏布局、表单行、列表项、危险区
├── js/
│   ├── storage.js          # 存储层：读写、默认种子、校验、迁移、导入导出（无 DOM 依赖）
│   ├── auth.js             # 会话层：登录/登出/有效期（无 DOM 依赖）
│   ├── render.js           # 视图层：数据 → 前台 DOM（只读存储）
│   └── admin.js            # 控制层：表单绑定、校验、项目 CRUD、备份
├── assets/
│   ├── avatar.svg          # 默认头像占位
│   └── favicon.svg         # 站点图标
├── PRD.md                  # 产品需求文档
├── TECH_DESIGN.md          # 技术设计文档
└── AGENTS.md               # 本文件
```

### 4.1 目录规则

- **扁平**：不建 `src/`、`dist/`、`components/` 等中间层。项目规模只要 2 页 + 4 个 JS。
- **JS 不拆更细**：不要为一个 20 行的工具函数新建文件。超过 400 行再考虑拆分。
- **样式按页面归属**：前台规则进 `site.css`，后台规则进 `admin.css`，两处都要用的进 `base.css`。不要建 `utilities.css`。
- **图片只放 `assets/`**：不要散落在根目录。
- **不要生成构建产物**：没有 `dist/`、没有 `.cache/`、没有锁文件。

### 4.2 文件职责边界

| 文件 | 可以做 | 不可以做 |
| --- | --- | --- |
| `storage.js` | 访问 `localStorage`、校验、序列化 | 操作 DOM、处理事件 |
| `auth.js` | 访问 `sessionStorage`、比对口令 | 操作 DOM |
| `render.js` | 读数据、生成 DOM、绑定展示 | 写存储 |
| `admin.js` | 读表单、调用 storage 写入、事件绑定 | 直接 `localStorage.setItem` |
| `index.html` | 引入脚本、声明挂载点 | 写业务逻辑 |

---

## 5. 数据契约（重要）

### 5.1 不可擅自变更

| 项 | 当前值 |
| --- | --- |
| 存储 key | `pw.profile.v1`、`pw.projects.v1`、`pw.meta.v1`、`pw.session` |
| 版本号 | `SCHEMA_VERSION = 1` |
| 字段名 | 见 `TECH_DESIGN.md` 第 4.2 节 |

**改动任何一项，必须同时做三件事**：

1. 递增 `SCHEMA_VERSION`，新增迁移函数；
2. 同步更新 `TECH_DESIGN.md` 第 4 节；
3. 同步更新 `PRD.md` 第 6 节的字段表。

否则会静默破坏站主已有的数据 —— 这是本项目最高优先级的一致性要求。

### 5.2 校验规则（改动需两侧同步）

- `name` 必填、≤ 24 字
- `headline` 必填、≤ 40 字
- `email` 必填、匹配 `/^[^\s@]+@[^\s@]+\.[^\s@]+$/`
- `title` / `summary`（项目）必填
- 所有 URL 字段仅允许 `http://` 或 `https://`

---

## 6. 编码约定

### 6.1 脚本加载方式（有原因，别改）

```html
<!-- index.html：按依赖顺序书写，defer 保证顺序执行 -->
<script src="js/storage.js" defer></script>
<script src="js/render.js" defer></script>
<!-- render.js 末尾自行调用 init()：DOMContentLoaded 后渲染 -->
```

```html
<!-- admin.html -->
<script src="js/storage.js" defer></script>
<script src="js/render.js" defer></script>
<script src="js/auth.js" defer></script>
<script src="js/admin.js" defer></script>
```

使用**普通脚本 + 全局命名空间**（`window.AppStorage` / `AppAuth` / `AppRender` / `AppAdmin`），不使用 ES Module。

> **原因**：`<script type="module">` 在 `file://` 协议下受 CORS 限制会被浏览器直接拒绝加载。为了让"双击 html 就能看到页面"成立，只能走普通脚本。`defer` 保证按书写顺序、在 DOM 解析完成后执行。

每个模块统一用 IIFE 包裹并暴露单个全局对象：

```js
(function (global) {
  'use strict';
  // ...
  global.AppXxx = { /* 公开 API */ };
})(window);
```

### 6.2 HTML

- 使用语义标签：`header` / `main` / `section` / `article` / `footer`。
- 行为钩子一律用 `data-*`：`data-mount`、`data-field`、`data-action`、`data-id`。
- **不写内联事件**（`onclick="…"`），统一 `addEventListener` + 事件委托。
- **不写内联样式**，颜色字号一律走 CSS 变量。
- 所有图片带 `alt`；纯装饰图写 `alt=""`。

### 6.3 CSS

- 类名 `block__element--modifier`（BEM-lite），全小写连字符。
- 颜色、间距、圆角、断点全部使用 `base.css` 中的变量，**不在业务样式里写死色值**。
- 禁止 `!important`；禁止 `#id` 选择器做样式。
- 单文件不超过 300 行，超过说明该拆或该简化。

### 6.4 JavaScript

- 严格模式；每个文件 IIFE。
- 缩进 2 空格，字符串用单引号，语句结尾带分号。
- 命名：函数 `camelCase`，常量 `UPPER_SNAKE`，DOM 引用变量以 `$` 前缀或不加（保持文件内一致）。
- 所有 `try/catch` 里必须 `console.warn` / `console.error` 带上 key 或场景名，禁止静默吞异常。
- 所有写 localStorage 的地方必须检查返回值，失败要反馈给用户。

### 6.5 安全

- 拼接 HTML 时**必须**转义：使用 `AppRender.esc()`。
- URL 必须经 `safeUrl()` 白名单过滤（只放行 `http/https/mailto`），防止 `javascript:` 注入。
- 外链一律 `target="_blank" rel="noopener noreferrer"`。

---

## 7. 常见任务怎么做

### 7.1 修改默认文案（首次访问展示的内容）

只改 `js/storage.js` 里的 `DEFAULT_PROFILE` / `DEFAULT_PROJECTS`。

⚠️ 注意：这些默认值**只在本地没有数据时生效**。如果站主已经录入过内容，改默认值不会生效，必须通过后台修改，或在后台"恢复默认内容"。

### 7.2 新增一个数据字段（如"微信号"）

按顺序做，一步都不能跳：

1. `storage.js`：`DEFAULT_PROFILE` 加字段 + `validateProfile` 加校验 + `SCHEMA_VERSION` 加 1 + 写迁移函数；
2. `TECH_DESIGN.md` 第 4.2 节更新 JSON 示例；
3. `PRD.md` 第 6.1 节字段表加一行；
4. `admin.html` 加输入框，绑 `data-field="wechat"`；
5. `admin.js` 的 `collectProfile()` / `fillProfile()` 补上该字段；
6. `render.js` 里决定是否展示、如何展示（不展示也要在 DOM 里留位置，避免布局塌陷）。

### 7.3 加一个项目

不需要改代码。打开后台 → 新增项目 → 填标题、摘要、链接 → 保存。

### 7.4 换主题色

只改 `css/base.css` 的 `--c-accent`（以及暗色模式下的同名变量），不要逐条改业务样式。

### 7.5 调整布局断点

统一在 `base.css` 定义，`site.css` / `admin.css` 复用相同的 `640px` / `960px` 两个断点。不要引入第三个断点。

---

## 8. 提交前自检清单

- [ ] 用 `http://localhost:8000` 打开 `index.html` 与 `admin.html`，控制台无报错、无 404。
- [ ] 首次访问（清空存储后）能正常显示默认内容，不白屏。
- [ ] 后台保存 → 前台刷新内容变化。
- [ ] 校验失败有明确提示，且**没有**写入坏数据。
- [ ] 没有引入任何外部网络请求（DevTools → Network 除本地文件外应为空）。
- [ ] 375px / 768px / 1440px 三档布局正常。
- [ ] 键盘可完成"登录 → 改姓名 → 保存"全流程。
- [ ] 改动数据结构时，`SCHEMA_VERSION`、三份文档同步更新。
- [ ] 没有 `console.log` 残留调试输出（`warn` / `error` 属于正常错误处理，可保留）。

---

## 9. 禁止事项（会被打回）

1. 为了让某段代码"更现代"而引入框架或构建链。
2. 直接把用户输入 `innerHTML` 拼接而不转义。
3. 在 `render.js` 里写数据、在 `admin.js` 里直接操作 `localStorage`，绕过 `storage.js`。
4. 修改 key 名或字段名却不做迁移（会让已有数据消失）。
5. 引入外部 CDN 字体 / 图标 / 统计代码。
6. 为简单功能新增 `utils/`、`helpers.js` 之类的抽象层。
7. 把后台口令当成真实安全机制来设计（它只是防误触）。
8. 新增第三个 HTML 页面来放"项目详情"（用展开交互解决）。

---

## 10. 三份文档的关系

```text
PRD.md          做什么、验收标准      ← 需求变更时先改这里
TECH_DESIGN.md  怎么实现、数据结构     ← 实现变更时同步这里
AGENTS.md       目录、约定、禁止事项   ← 协作规则变更时同步这里
```

三者出现矛盾时：**以 `PRD.md` 的功能意图为准**，但数据结构与 key 一律以 `TECH_DESIGN.md` 第 4 节为准。发现不一致，先修文档再写代码。
