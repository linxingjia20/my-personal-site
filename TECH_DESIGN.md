# TECH DESIGN · 纯静态个人网站 + localStorage 后台

| 项目 | 内容 |
| --- | --- |
| 文档版本 | v1.0 |
| 更新日期 | 2026-10-20 |
| 上游文档 | `PRD.md`（功能与验收）、`AGENTS.md`（目录与协作约定） |
| 设计原则 | 零构建、零依赖、零后端；两个 HTML 页面即可交付 |

---

## 1. 设计目标与约束

1. **零后端**：不写服务端代码，不上数据库，不需要部署运行时。
2. **零构建**：没有 `npm install`、没有打包器、没有编译步骤；改完文件刷新浏览器即生效。
3. **单机可迁移**：内容存在浏览器本地，通过 JSON 文件导出 / 导入实现搬家和备份。
4. **直白优先**：能用 40 行原生 JS 解决的事情不引入框架；代码可读性 > 抽象层级。

**因此明确放弃**：Vue / React、Vite / Webpack、TypeScript 编译、SCSS、CSS 框架、任何 CDN 资源。

---

## 2. 技术选型

| 关注点 | 选择 | 理由 | 放弃的方案 |
| --- | --- | --- | --- |
| 页面结构 | 原生 HTML5，两个静态页 | 无需路由，SEO 与加载都最优 | SPA 路由 |
| 样式 | 原生 CSS + CSS 变量 | 主题色集中管理，可读性高 | Tailwind / Bootstrap |
| 逻辑 | 原生 JavaScript（ES2020） | 无依赖、无编译 | 任何框架 |
| 脚本加载 | 普通 `<script defer>` + 全局命名空间 | **`type="module"` 在 `file://` 下会被 CORS 拦截**，用普通脚本才能双击打开 | ES Module |
| 持久化 | `localStorage` | 同源持久、API 简单、同步读取方便首屏渲染 | IndexedDB（异步、复杂）、后端 DB |
| 会话 | `sessionStorage` | 关掉标签页即失效，符合后台使用习惯 | cookie |
| 图片 | 前端 canvas 压缩后转 dataURL | 避免 storage 被原图撑爆 | 原图存 storage |

---

## 3. 总体架构

```text
                    ┌──────────────────────────────────────────┐
                    │              浏览器（同源）                │
                    │                                          │
  访客 ───────────▶ │  index.html ──┐                          │
                    │    render.js ◀┤                          │
                    │               │ 读                        │
                    │  admin.html   │                          │
  站主 ───────────▶ │    auth.js    │                          │
                    │    admin.js ◀─┼──▶ storage.js ──▶ localStorage │
                    │               │       ▲                  │
                    │               │       │ 写                │
                    │   （导入/导出）─┴───────┘                  │
                    └──────────────────────────────────────────┘
                                      │
                                      ▼
                            导出文件 data.json
                            （站主手动备份 / 迁移）
```

**数据流要点**

- **唯一写入方**：只有 `admin.js`（以及 `storage.js` 内部的初始化）会调用写操作；前台只读。
- **唯一存储入口**：所有对 `localStorage` 的访问都封装在 `storage.js`，其他模块禁止直接调用。
- **渲染单向**：`storage → render → DOM`，`render.js` 不写存储，不做业务判断。

---

## 4. 存储设计

### 4.1 Key 规划

| Key | 载体 | 内容 | 说明 |
| --- | --- | --- | --- |
| `pw.profile.v1` | localStorage | Profile 对象 | 个人信息 |
| `pw.projects.v1` | localStorage | Project[] 数组 | 项目列表，数组顺序即展示顺序 |
| `pw.meta.v1` | localStorage | `{ schemaVersion, updatedAt }` | 版本号与最近更新时间 |
| `pw.session` | sessionStorage | `{ at: 时间戳 }` | 登录会话 |

> 命名规范：`pw.`（personal website）前缀 + 语义名 + `.v1` 版本后缀。**改结构时递增版本号，不要复用旧 key。**

### 4.2 数据结构

```jsonc
// pw.profile.v1
{
  "name": "张三",
  "headline": "前端工程师 / 独立开发者",
  "tagline": "做让人愿意多看一眼的界面。",
  "bio": "第一段简介。\n\n第二段简介。",
  "avatar": "",
  "location": "中国 · 杭州",
  "email": "hi@example.com",
  "skills": ["HTML", "CSS", "JavaScript"],
  "socials": [
    { "label": "GitHub", "url": "https://github.com/yourname" }
  ]
}
```

```jsonc
// pw.projects.v1
[
  {
    "id": "p-1729500000000",
    "title": "示例项目",
    "summary": "一句话说明这个项目解决了什么问题。",
    "description": "可选的长描述。",
    "tags": ["JavaScript"],
    "link": "",
    "repo": "",
    "cover": "",
    "featured": true
  }
]
```

```jsonc
// pw.meta.v1
{ "schemaVersion": 1, "updatedAt": "2026-10-20T00:00:00.000Z" }
```

字段含义、必填与长度限制以 `PRD.md` 第 6 节为准，两侧必须同步修改。

### 4.3 版本与迁移

- 常量 `SCHEMA_VERSION` 在 `storage.js` 顶部定义，当前为 `1`。
- 读取时若 `meta.schemaVersion < SCHEMA_VERSION`，执行迁移函数并回写。
- 迁移函数用**函数映射表**而非一长串 `if`，便于逐版追加：

```js
const MIGRATIONS = {
  // 1 → 2 示例：给老数据补字段
  // 1: (state) => ({ ...state, profile: { ...state.profile, newField: '' } }),
};
```

- 迁移失败时不写入，保留原始数据并在控制台报错，避免"改坏后无法回滚"。

### 4.4 写入策略

1. 先构造完整对象并 `JSON.stringify`（可能抛错，如循环引用），**序列化成功后再写**，避免写进去半个坏值。
2. `setItem` 用 `try/catch` 包裹，捕获 `QuotaExceededError` / 隐私模式禁用等情况，返回布尔值供 UI 提示。
3. 每次成功写入后更新 `pw.meta.v1.updatedAt`。
4. 不做"自动保存"，全部由用户点按钮触发，避免半成品数据落盘。

### 4.5 头像处理

直接用 `localStorage` 存原图必然爆容量，流程如下：

```js
async function compressImage(file, maxEdge = 512, quality = 0.85) {
  const bitmap = await createImageBitmap(file);            // 需要 https / localhost
  const scale = Math.min(1, maxEdge / Math.max(bitmap.width, bitmap.height));
  const w = Math.round(bitmap.width * scale);
  const h = Math.round(bitmap.height * scale);
  const canvas = document.createElement('canvas');
  canvas.width = w;
  canvas.height = h;
  canvas.getContext('2d').drawImage(bitmap, 0, 0, w, h);
  return canvas.toDataURL('image/jpeg', quality);           // 一般 < 200KB
}
```

- 限制：单张头像处理结果 > 400KB 时拒绝并提示。
- `createImageBitmap` 不可用时（极老浏览器），降级为 `FileReader` 直读并提示尺寸可能过大。

---

## 5. 模块设计

```text
js/
├── storage.js   存储层：读写 + 默认种子 + 校验 + 迁移 + 导入导出   （无 DOM 依赖）
├── auth.js      会话层：登录 / 登出 / 会话有效期判断                （无 DOM 依赖）
├── render.js    视图层：把数据渲染成前台 DOM                       （只读存储）
└── admin.js     控制层：表单绑定、校验、项目 CRUD、备份             （读写存储）
```

### 5.1 `storage.js` 对外 API

| 方法 | 入参 | 返回 | 说明 |
| --- | --- | --- | --- |
| `getProfile()` | — | Profile | 不存在时返回默认种子 |
| `saveProfile(p)` | Profile | `boolean` | 写前校验，失败返回 false |
| `getProjects()` | — | Project[] | 不存在时返回默认数组 |
| `saveProjects(list)` | Project[] | `boolean` | 整体覆盖写 |
| `resetAll()` | — | `void` | 恢复默认内容 |
| `exportJSON()` | — | `string` | 生成备份文本 |
| `importJSON(text)` | string | `{ ok, error? }` | 校验 + 覆盖 |
| `validateProfile(p)` | Profile | `string[]` | 返回错误文案数组 |
| `validateProject(p)` | Project | `string[]` | 同上 |
| `newProjectId()` | — | string | `p-` + 时间戳 |

### 5.2 依赖关系

```text
index.html → storage.js → render.js
admin.html → storage.js → auth.js → admin.js
```

无循环依赖，`storage.js` 与 `auth.js` 都不碰 DOM，可单独在控制台调试。

### 5.3 DOM 约定

JS 通过 `data-*` 属性定位元素，**不使用 class 做行为选择器**（class 归样式所有，互不污染）：

| 属性 | 用途 | 示例 |
| --- | --- | --- |
| `data-field="name"` | 表单项映射到数据字段 | `<input data-field="name">` |
| `data-mount="hero"` | 渲染挂载点 | `<header data-mount="hero">` |
| `data-action="save"` | 事件委托的处理动作 | `<button data-action="save">` |
| `data-id="p-1729…"` | 列表项绑定的项目 id | 用于上移 / 删除 |

---

## 6. 关键流程

### 6.1 首次渲染（前台）

```text
浏览器解析 HTML
  └─ defer 脚本依次执行：storage.js → render.js
       └─ render.init()
            ├─ 读 pw.profile.v1 ── 无 ─┐
            ├─ 读 pw.projects.v1 ── 无 ─┤→ 使用内置 DEFAULT_* 并写入（初始化一次）
            ├─ 校验失败 / JSON 非法 ───┘→ 降级默认 + console.warn
            ├─ 渲染 Hero（头像为空 → 首字母占位）
            ├─ 渲染 关于我（bio 按 \n\n 分段，skills 渲染为标签）
            ├─ 渲染 项目（featured 优先，其余按数组顺序；空数组 → 空态）
            └─ 渲染 页脚（年份自动取当前年）
```

### 6.2 保存个人信息（后台）

```text
点击 [保存修改]
  └─ admin.collectForm()  → 从 [data-field] 读取并整理成 Profile
       └─ storage.validateProfile() → 有错误：逐项标红 + 聚焦首个错误 + return
            └─ storage.saveProfile() → false：toast「存储失败：空间不足」+ return
                 └─ 成功：更新 meta.updatedAt + toast「已保存」+ 清除 dirty 标记
```

### 6.3 登录

```text
访问 admin.html
  └─ auth.isLoggedIn()
       ├─ 否 → 只渲染登录卡片，管理区 display:none
       └─ 是 → 渲登录卡片隐藏，管理区显示，admin.init()
提交口令
  └─ auth.login(input) → 比对常量 → 成功写 sessionStorage → 重渲染界面
退出
  └─ auth.logout() → 清 sessionStorage → location.reload()
```

---

## 7. 核心代码骨架

### 7.1 `js/storage.js`

```js
/* 唯一允许访问 localStorage 的模块。无 DOM 依赖。 */
(function (global) {
  'use strict';

  const SCHEMA_VERSION = 1;

  const KEYS = {
    profile: 'pw.profile.v1',
    projects: 'pw.projects.v1',
    meta: 'pw.meta.v1',
    session: 'pw.session',
  };

  const DEFAULT_PROFILE = {
    name: '你的名字',
    headline: '前端工程师 / 独立开发者',
    tagline: '做让人愿意多看一眼的界面。',
    bio: '这里是第一段简介：我在做什么、擅长什么。\n\n这里是第二段：最近在关注什么，欢迎交流。',
    avatar: '',
    location: '中国 · 杭州',
    email: 'hi@example.com',
    skills: ['HTML', 'CSS', 'JavaScript'],
    socials: [{ label: 'GitHub', url: 'https://github.com/yourname' }],
  };

  const DEFAULT_PROJECTS = [
    {
      id: 'p-demo-1',
      title: '示例项目',
      summary: '一句话说明这个项目解决了什么问题。',
      description: '',
      tags: ['JavaScript'],
      link: '',
      repo: '',
      cover: '',
      featured: true,
    },
  ];

  /* ---------- 底层读写 ---------- */

  function read(key, fallback) {
    try {
      const raw = global.localStorage.getItem(key);
      return raw == null ? fallback : JSON.parse(raw);
    } catch (err) {
      console.warn('[storage] 读取失败，已降级为默认值：', key, err);
      return fallback;
    }
  }

  function write(key, value) {
    try {
      const raw = JSON.stringify(value); // 先序列化，失败则不落盘
      global.localStorage.setItem(key, raw);
      return true;
    } catch (err) {
      console.error('[storage] 写入失败：', key, err);
      return false;
    }
  }

  function touchMeta() {
    write(KEYS.meta, { schemaVersion: SCHEMA_VERSION, updatedAt: new Date().toISOString() });
  }

  /* ---------- 校验 ---------- */

  const isStr = (v) => typeof v === 'string';
  const isUrl = (v) => /^https?:\/\/\S+$/i.test(v);

  function validateProfile(p) {
    const errors = [];
    if (!p || !isStr(p.name) || !p.name.trim()) errors.push('姓名不能为空');
    if (!isStr(p.headline) || !p.headline.trim()) errors.push('头衔不能为空');
    if (!isStr(p.email) || !/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(p.email)) {
      errors.push('邮箱格式不正确');
    }
    if (p.name && p.name.length > 24) errors.push('姓名不能超过 24 个字');
    (p.socials || []).forEach((s, i) => {
      if (s.url && !isUrl(s.url)) errors.push(`第 ${i + 1} 条社交链接不是合法的 http(s) 地址`);
    });
    return errors;
  }

  function validateProject(pr) {
    const errors = [];
    if (!pr || !isStr(pr.title) || !pr.title.trim()) errors.push('项目名称不能为空');
    if (!isStr(pr.summary) || !pr.summary.trim()) errors.push('项目摘要不能为空');
    if (pr.link && !isUrl(pr.link)) errors.push('项目链接不是合法的 http(s) 地址');
    if (pr.repo && !isUrl(pr.repo)) errors.push('仓库链接不是合法的 http(s) 地址');
    return errors;
  }

  /* ---------- 对外 API ---------- */

  function getProfile() {
    const p = read(KEYS.profile, null);
    return p && validateProfile(p).length === 0 ? p : { ...DEFAULT_PROFILE };
  }

  function saveProfile(p) {
    if (validateProfile(p).length) return false;
    if (!write(KEYS.profile, p)) return false;
    touchMeta();
    return true;
  }

  function getProjects() {
    const list = read(KEYS.projects, null);
    return Array.isArray(list) ? list : DEFAULT_PROJECTS.map((x) => ({ ...x }));
  }

  function saveProjects(list) {
    if (!Array.isArray(list)) return false;
    if (!write(KEYS.projects, list)) return false;
    touchMeta();
    return true;
  }

  function newProjectId() {
    return 'p-' + Date.now();
  }

  function ensureSeeded() {
    if (global.localStorage.getItem(KEYS.profile) == null) write(KEYS.profile, DEFAULT_PROFILE);
    if (global.localStorage.getItem(KEYS.projects) == null) write(KEYS.projects, DEFAULT_PROJECTS);
    if (global.localStorage.getItem(KEYS.meta) == null) touchMeta();
  }

  function resetAll() {
    [KEYS.profile, KEYS.projects, KEYS.meta].forEach((k) => global.localStorage.removeItem(k));
    ensureSeeded();
  }

  function exportJSON() {
    return JSON.stringify(
      {
        app: 'personal-website',
        schemaVersion: SCHEMA_VERSION,
        exportedAt: new Date().toISOString(),
        profile: getProfile(),
        projects: getProjects(),
      },
      null,
      2
    );
  }

  function importJSON(text) {
    let data;
    try {
      data = JSON.parse(text);
    } catch {
      return { ok: false, error: '文件不是合法的 JSON' };
    }
    if (data.app !== 'personal-website') return { ok: false, error: '不是本站的备份文件' };
    if (data.schemaVersion > SCHEMA_VERSION) return { ok: false, error: '备份版本高于当前版本' };
    const errs = validateProfile(data.profile);
    if (errs.length) return { ok: false, error: '备份内容有问题：' + errs.join('；') };
    if (!Array.isArray(data.projects)) return { ok: false, error: '备份缺少项目列表' };
    if (!write(KEYS.profile, data.profile)) return { ok: false, error: '写入失败，可能是空间不足' };
    if (!write(KEYS.projects, data.projects)) return { ok: false, error: '写入失败，可能是空间不足' };
    touchMeta();
    return { ok: true };
  }

  global.AppStorage = {
    KEYS, SCHEMA_VERSION, DEFAULT_PROFILE, DEFAULT_PROJECTS,
    getProfile, saveProfile, getProjects, saveProjects, newProjectId,
    ensureSeeded, resetAll, exportJSON, importJSON,
    validateProfile, validateProject,
  };
})(window);
```

### 7.2 `js/auth.js`

```js
/* 本地口令门。注意：口令写在前端，绕过成本为零，仅用于防误操作。 */
(function (global) {
  'use strict';

  const PASSWORD = '123456';                // 建议首次使用后自行修改
  const TTL = 30 * 60 * 1000;               // 会话有效期：30 分钟
  const KEY = 'pw.session';

  function isLoggedIn() {
    try {
      const raw = global.sessionStorage.getItem(KEY);
      if (!raw) return false;
      const { at } = JSON.parse(raw);
      if (!at || Date.now() - at > TTL) return (logout(), false);
      return true;
    } catch {
      return false;
    }
  }

  function login(input) {
    if (input !== PASSWORD) return false;
    try {
      global.sessionStorage.setItem(KEY, JSON.stringify({ at: Date.now() }));
    } catch {
      /* 隐私模式下 sessionStorage 可能不可用，降级为本次页面内有效 */
    }
    return true;
  }

  function logout() {
    try { global.sessionStorage.removeItem(KEY); } catch { /* 忽略 */ }
  }

  global.AppAuth = { isLoggedIn, login, logout };
})(window);
```

### 7.3 `js/render.js`（要点）

```js
(function (global) {
  'use strict';

  const ESC = { '&': '&amp;', '<': '&lt;', '>': '&gt;', '"': '&quot;', "'": '&#39;' };
  const esc = (s) => String(s ?? '').replace(/[&<>"']/g, (c) => ESC[c]);

  // 只放行 http/https/mailto，阻断 javascript: 之类的危险协议
  function safeUrl(u) {
    const v = String(u || '').trim();
    return /^(https?:\/\/|mailto:)/i.test(v) ? v : '';
  }

  function renderHero(profile) {
    const mount = document.querySelector('[data-mount="hero"]');
    const avatar = profile.avatar
      ? `<img class="avatar" src="${esc(safeUrl(profile.avatar) || profile.avatar)}" alt="${esc(profile.name)} 的头像">`
      : `<div class="avatar avatar--initial" aria-hidden="true">${esc((profile.name || '?').slice(0, 1))}</div>`;
    mount.innerHTML = `
      ${avatar}
      <div class="hero__text">
        <h1>${esc(profile.name)}</h1>
        <p class="hero__headline">${esc(profile.headline)}</p>
        ${profile.tagline ? `<p class="hero__tagline">${esc(profile.tagline)}</p>` : ''}
        ${renderSocials(profile.socials)}
      </div>`;
  }

  function renderSocials(socials) {
    const items = (socials || []).filter((s) => safeUrl(s.url));
    if (!items.length) return '';
    return `<ul class="socials">${items
      .map((s) => `<li><a href="${esc(safeUrl(s.url))}" target="_blank" rel="noopener noreferrer">${esc(s.label || s.url)}</a></li>`)
      .join('')}</ul>`;
  }

  function renderProjects(list) {
    const mount = document.querySelector('[data-mount="projects"]');
    if (!list.length) {
      mount.innerHTML = `<p class="empty">作品整理中，敬请期待。</p>`;
      return;
    }
    // 置顶优先，其余保持数组顺序（Object.assign 稳定排序）
    const ordered = list
      .map((p, i) => ({ p, i }))
      .sort((a, b) => Number(b.p.featured) - Number(a.p.featured) || a.i - b.i)
      .map((x) => x.p);

    mount.innerHTML = ordered.map(projectCard).join('');
  }

  function projectCard(p) {
    const cover = p.cover
      ? `<img class="card__cover" src="${esc(safeUrl(p.cover) || p.cover)}" alt="" loading="lazy">`
      : `<div class="card__cover card__cover--blank" aria-hidden="true"></div>`;
    const tags = (p.tags || []).map((t) => `<li class="tag">${esc(t)}</li>`).join('');
    const link = safeUrl(p.link) || safeUrl(p.repo);
    const title = link
      ? `<a href="${esc(link)}" target="_blank" rel="noopener noreferrer">${esc(p.title)}</a>`
      : esc(p.title);
    return `
      <article class="card">
        ${cover}
        <h3 class="card__title">${title}</h3>
        <p class="card__summary">${esc(p.summary)}</p>
        ${tags ? `<ul class="tags">${tags}</ul>` : ''}
      </article>`;
  }

  function init() {
    global.AppStorage.ensureSeeded();
    const profile = global.AppStorage.getProfile();
    renderHero(profile);
    renderAbout(profile);
    renderProjects(global.AppStorage.getProjects());
    document.querySelectorAll('[data-year]').forEach((el) => (el.textContent = new Date().getFullYear()));
  }

  global.AppRender = { init, esc, safeUrl };
})(window);
```

> `renderAbout` 同理：`bio.split(/\n{2,}/)` 逐段包 `<p>`，`skills` 渲染为标签列表。

### 7.4 `js/admin.js`（要点）

```js
(function (global) {
  'use strict';

  const S = global.AppStorage;
  const R = global.AppRender;
  let dirty = false;

  /* 表单 → 数据 */
  function collectProfile() {
    const val = (f) => document.querySelector(`[data-field="${f}"]`)?.value.trim() ?? '';
    return {
      name: val('name'),
      headline: val('headline'),
      tagline: val('tagline'),
      bio: document.querySelector('[data-field="bio"]').value.trim(),
      avatar: val('avatar'),                 // 隐藏域，由上传逻辑写入 dataURL
      location: val('location'),
      email: val('email'),
      skills: val('skills').split(/[,，]/).map((s) => s.trim()).filter(Boolean),
      socials: collectSocials(),             // [{label,url}]
    };
  }

  function fillProfile(p) {
    Object.entries(p).forEach(([k, v]) => {
      const el = document.querySelector(`[data-field="${k}"]`);
      if (!el) return;
      el.value = Array.isArray(v) ? v.map((x) => (typeof x === 'string' ? x : x.url)).join(', ') : v;
    });
  }

  /* 保存 */
  function onSave() {
    const profile = collectProfile();
    const errors = S.validateProfile(profile);
    if (errors.length) return toast(errors[0], 'error');
    if (!S.saveProfile(profile)) return toast('保存失败：存储空间可能已满', 'error');
    dirty = false;
    toast('已保存');
  }

  /* 项目 CRUD：整体读 → 改数组 → 整体写 */
  function upsertProject(p) {
    const list = S.getProjects();
    const idx = list.findIndex((x) => x.id === p.id);
    if (idx >= 0) list[idx] = p; else list.push({ ...p, id: p.id || S.newProjectId() });
    if (!S.saveProjects(list)) return toast('保存失败：存储空间可能已满', 'error');
    renderProjectList();
    toast('项目已保存');
  }

  function moveProject(id, delta) {
    const list = S.getProjects();
    const i = list.findIndex((x) => x.id === id);
    const j = i + delta;
    if (i < 0 || j < 0 || j >= list.length) return;
    [list[i], list[j]] = [list[j], list[i]];
    S.saveProjects(list);
    renderProjectList();
  }

  function removeProject(id) {
    if (!confirm('删除后不可恢复，确定删除该项目？')) return;
    S.saveProjects(S.getProjects().filter((x) => x.id !== id));
    renderProjectList();
  }

  /* 备份 */
  function exportBackup() {
    const blob = new Blob([S.exportJSON()], { type: 'application/json' });
    const a = document.createElement('a');
    a.href = URL.createObjectURL(blob);
    a.download = `personal-website-${new Date().toISOString().slice(0, 10)}.json`;
    a.click();
    URL.revokeObjectURL(a.href);
  }

  function importBackup(file) {
    const reader = new FileReader();
    reader.onload = () => {
      const res = S.importJSON(String(reader.result));
      if (!res.ok) return toast(res.error, 'error');
      toast('导入成功');
      setTimeout(() => location.reload(), 600);
    };
    reader.readAsText(file);
  }

  global.AppAdmin = { collectProfile, fillProfile, onSave, upsertProject, moveProject, removeProject, exportBackup, importBackup };
  global.addEventListener('beforeunload', (e) => { if (dirty) e.preventDefault(); });
})(window);
```

---

## 8. 样式架构

### 8.1 文件划分

```text
css/
├── base.css    reset / CSS 变量 / 排版 / 通用组件（按钮、输入框、标签、toast）
├── site.css    前台专用（hero、about、cards、footer）
└── admin.css   后台专用（布局、表单行、列表项、危险区）
```

### 8.2 设计令牌（`base.css`）

```css
:root {
  --c-bg: #ffffff;
  --c-surface: #f6f7f8;
  --c-text: #1a1a1a;
  --c-muted: #6b7280;
  --c-border: #e5e7eb;
  --c-accent: #2563eb;
  --c-danger: #dc2626;

  --space-1: 0.25rem;  /* 4px  */
  --space-2: 0.5rem;   /* 8px  */
  --space-3: 1rem;     /* 16px */
  --space-4: 1.5rem;   /* 24px */
  --space-5: 2.5rem;   /* 40px */
  --space-6: 4rem;     /* 64px */

  --radius: 10px;
  --maxw: 960px;
  --font: system-ui, -apple-system, "Segoe UI", "Microsoft YaHei", sans-serif;
}

@media (prefers-color-scheme: dark) {
  :root {
    --c-bg: #0f1115;
    --c-surface: #171a21;
    --c-text: #e8eaed;
    --c-muted: #9aa0a6;
    --c-border: #262a33;
    --c-accent: #60a5fa;
  }
}
```

### 8.3 断点与规则

| 断点 | 项目网格 | 后台布局 |
| --- | --- | --- |
| ≥ 960px | 3 列 | 表单 + 预览双栏 |
| 640–959px | 2 列 | 单栏，预览折叠到顶部 |
| < 640px | 1 列 | 单栏，按钮通栏 |

- 网格实现：`grid-template-columns: repeat(auto-fill, minmax(260px, 1fr))`。
- 禁止 `!important`；禁止内联 `style`（动态 computed 值除外）。

---

## 9. 错误处理与降级

| 场景 | 处理 |
| --- | --- |
| `localStorage` 被禁用（隐私模式 / 策略） | 检测写入失败，页面顶部横幅提示"内容无法保存"，前台仍以默认数据正常展示 |
| 存储的 JSON 损坏 | `read()` 捕获异常 → 返回 fallback → `console.warn` → 页面不白屏 |
| 写入超出配额 | `write()` 返回 false → toast 明确提示 → 提示压缩头像或删项目 |
| 图片加载失败 | `onerror` 替换为占位块，避免破图图标 |
| 头像 URL 非法协议 | `safeUrl()` 返回空 → 走首字母占位 |
| 导入文件格式错误 | 返回具体原因（非 JSON / 非本站备份 / 版本过高） |

---

## 10. 测试方案

### 10.1 手工用例

| # | 前置 | 步骤 | 预期 |
| --- | --- | --- | --- |
| 1 | 清空存储 | 打开 `index.html` | 显示默认姓名与 1 个示例项目，控制台无报错 |
| 2 | 用例 1 之后 | 直接打开 `admin.html` | 只显示登录门 |
| 3 | 已登录 | 改姓名 → 保存 → 打开前台 | 姓名已更新 |
| 4 | 已登录 | 姓名清空 → 保存 | 阻止保存，输入框下方红字提示 |
| 5 | 已登录 | 新增 2 个项目 → 上移 | 顺序变化，前台同步 |
| 6 | 已登录 | 删除项目 → 取消确认 | 项目保留 |
| 7 | 已登录 | 导出 → 恢复默认 → 导入 | 内容完整还原 |
| 8 | — | 存 `localStorage.setItem('pw.profile.v1','{bad')` 后刷新 | 降级默认，不白屏 |
| 9 | 上传 3MB 图片 | 压缩后保存 | 成功且存储占用 < 400KB |
| 10 | 375px 视口 | 浏览前台 | 单列，无横向滚动 |
| 11 | 键盘 | Tab 遍历前台与后台 | 焦点顺序合理，焦点样式可见 |

### 10.2 自动化（可选）

不引入测试框架。如需回归校验，可写一个 `node scripts/check.mjs`：用 `node --experimental-...` 无关，直接在 Node 里 `import` 一份数据 JSON 并复用 `validateProfile` 的纯函数逻辑（需把校验函数抽成不依赖 `window` 的纯函数）。

---

## 11. 部署与"数据会出现不一致"的说明

- 可部署到 GitHub Pages / Netlify / 任意静态托管，纯文件上传即可。
- **关键提醒**：`localStorage` 按"协议 + 域名 + 端口"隔离。因此
  - `file://` 打开、`http://localhost:8000`、`https://yourname.github.io` 是**三份互不相通的数据**。
  - 上线后需要在线上重新录入内容，或用导出的 JSON 导入一次。
  - 日常维护请固定使用同一个访问地址（建议长期用线上地址）。

---

## 12. 演进路线

| 版本 | 方向 | 做法 |
| --- | --- | --- |
| v1.1 | 项目详情 | 卡片点击展开长描述，不新增页面 |
| v1.2 | 主题切换 | 增加手动 `data-theme` 覆盖系统偏好 |
| v2.0 | 内容进仓库 | 把 `DEFAULT_*` 换成 `content.json`，用 `fetch` 读取；后台改为"生成 JSON 供手动提交" |
| v2.1 | 多端同步 | 接入 Supabase / Firebase（真正的服务端存储），`storage.js` 换成异步适配层，UI 层不变 |
| v2.2 | 博客 | 独立 `notes.html`，Markdown 在构建期或运行期渲染 |

**演进前提**：`storage.js` 是唯一数据出入口，替换实现时只需保持第 5.1 节的 API 签名不变。
