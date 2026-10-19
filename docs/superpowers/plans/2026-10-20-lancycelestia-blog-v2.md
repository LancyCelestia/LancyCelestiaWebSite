# 澜汐斋记 实施计划 v2

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.
>
> **v1 已废弃**（`2026-10-20-lancycelestia-blog-v1-superseded.md`），原因是三份独立对抗审查共发现 69 个问题，其中 2 个阻塞级。本版逐条消化。

**Goal:** 在 `blog.lancycelestia.com` 上线一个客户端渲染四种视觉风格、服务器零渲染、中英双语的个人博客，并完成 ICP 备案；`www` 与裸域各放一份页面副本（不跳转）。

**Architecture:** 构建期用 Astro 7 生成纯静态产物，本地 Windows 构建 + Brotli 预压（**仅文本白名单**），`tar+scp` 原子发布，Nginx 只发文件。后端两个独立进程：`Artalk`（评论/注册/登录）与 `submitd`（自研投稿，Go + SQLite），分别反代到 `/comments/*` 与 `/api/submit/*`。所有动效客户端 GPU 渲染。

**Tech Stack:** Astro 7.4.1 / Node ≥22.22.0 / 原生 CSS / TypeScript / Nginx（brotli_static）/ Artalk / Go 1.x + modernc.org/sqlite / certbot / Playwright / webwallgl v3.0.0（按需）

**Spec:** `docs/superpowers/specs/2026-10-20-lancycelestia-blog-design.md`

---

## Global Constraints

1. 主域 `blog.lancycelestia.com`；`www.` 与裸域**各放一份页面副本，不做任何 301/302**
2. 备案网站名称：**澜汐斋记**
3. 站内展示名：潮声茶馆 / 澜汐 / 霞月 / LancyCelestia
4. Node **>=22.22.0**；本机 v26.7.0
5. **禁止**字体 `Inter`/`Roboto`/`Arial`/`Helvetica`/`Open Sans`（判定方式见下，**禁止用裸 grep**）
6. **禁止** `font-display: swap`；拉丁 Web 字体必须 `optional` + `size-adjust`
7. 中文**不加载 Web 字体**，走系统栈
8. `backdrop-filter` **只用于 fixed/sticky 元素与小型浮层**（Nav、搜索弹层、设置面板）。**滚动文档流内的卡片一律用 Mica（不用 backdrop）**
9. 动效只允许 `transform` / `opacity`；**唯一例外**：签名元素「潮线」用 `stroke-dashoffset`，需在 Task 9 显式记录豁免
10. 三档动效键名 `data-motion="low|mid|high"`，**默认 `low`**，由 JS 升档
11. `prefers-reduced-motion: reduce` → 在 **CSS 层**强制静止（不依赖 JS）
12. 构建产物含 `.br` 与 `.gz`，**仅对文本扩展名**：`html css js mjs svg xml json txt webmanifest`
13. **不得**打包 Wallpaper Engine 创意工坊资源（`.pkg/.tex/.webm/.mp3/.wav`）
14. 服务器侧操作由**用户**执行脚本，Agent 不持有 SSH 私钥

### 测试体系铁律（v1 的最大失败点）

> v1 有 12 个 `check-*.mjs` 全是对**源码字符串**做 `includes`/正则。删掉 `Base.astro` 的 `import '../styles/mica.css'` 后 Mica 完全不可见，而所有测试仍全绿。

**本版规定：**

- **T1｜禁止对源码做字符串包含断言**。要验证行为，就验证**构建产物**（解析 `dist/**/*.html` 的 DOM）或**运行时**（Playwright）。
- **T2｜每个视觉特性必须有一条"删掉它就变红"的测试**。写完断言后，**故意把实现删掉跑一次**，必须失败。这一步写进每个相关任务的 Step。
- **T3｜断言必须由同一进程的 Node 脚本执行**，用 `node:fs` + 正则/DOM 解析，**不得**用 shell `grep` 管道（`grep -E "\|"` 是字面竖线，v1 因此在验收里写了三条恒真断言）。
- **T4｜import 链必须被断言**。每个 CSS/组件都要有一条断言证明它**真的进了构建产物**（检查 `dist/_astro/*.css` 的合并结果，而非源文件）。
- **T5｜禁止 `|| echo`** 出现在任何断言里；断言脚本必须以非零码退出。

## Review Focus

1. **首帧主题闪烁**：`<head>` 内联脚本必须在首次绘制前写入 `data-theme`；且 `localStorage` 被禁用时不得抛错中断。
2. **无 `backdrop-filter` 支持**：面板必须回落到不透明底色，正文可读。
3. **`prefers-reduced-motion`**：渐变/潮线/滚动进场全部静止。
4. **后端全挂**：文章页正文正常渲染，评论区显示「评论暂不可用」，投稿页显示「服务暂不可用」。
5. **无 JS / 爬虫**：正文、标题、导航都在 HTML 里。
6. **`www` / 裸域**：可访问且内容与 `blog.` 一致，**但不得返回 301/302 到 `blog.`**。

---

## 文件结构

```
lancycelestia.com/
├─ astro.config.mjs
├─ playwright.config.ts
├─ .gitignore
├─ src/
│  ├─ config/site.ts             # 站点元信息 + features 开关（唯一真源）
│  ├─ i18n/ui.ts                 # 中英文案唯一真源
│  ├─ styles/
│  │  ├─ tokens.css              # 颜色/字阶/动效变量
│  │  ├─ glass.css               # 液态玻璃（仅浮层）
│  │  ├─ mica.css                # 云母质感（卡片，无 backdrop）
│  │  └─ global.css              # 重置 + :focus-visible + reduced-motion
│  ├─ components/
│  │  ├─ AuroraBackground.astro  # 动态釉瑚渐变（无 filter:blur）
│  │  ├─ GlassPanel.astro        # 浮层玻璃（Double-Bezel）
│  │  ├─ MicaCard.astro          # 滚动流卡片（无 backdrop）
│  │  ├─ TideLine.astro          # 签名元素
│  │  ├─ ThemeToggle.astro  MotionSwitch.astro
│  │  ├─ Wallpaper.astro         # 按需加载 webwallgl
│  │  ├─ Nav.astro  Footer.astro  SearchModal.astro  LangSwitch.astro
│  │  └─ Comments.astro  SubmitForm.astro
│  ├─ layouts/Base.astro
│  ├─ pages/
│  │  ├─ index.astro  about.astro  links.astro  projects/index.astro
│  │  ├─ blog/index.astro  blog/[...slug].astro  blog/page/[page].astro
│  │  ├─ tags/index.astro  tags/[tag].astro  submit.astro
│  │  ├─ en/…（镜像）
│  │  ├─ rss.xml.ts  404.astro
│  └─ content/blog/  content/projects/
├─ server/
│  └─ submitd/                   # 自研投稿服务
│     ├─ main.go  store.go  handlers.go  store_test.go  handlers_test.go
│     └─ go.mod
├─ scripts/
│  ├─ compress.mjs               # 文本白名单预压
│  ├─ deploy.sh
│  └─ assert/                    # 所有断言脚本（Node，非 grep）
│     ├─ fonts.mjs  tokens.mjs  glass.mjs  motion.mjs
│     ├─ content.mjs  i18n.mjs  search.mjs  assets.mjs
│     ├─ compress.mjs  imports.mjs
├─ tests/e2e/                    # Playwright 行为测试
│  ├─ visual.spec.ts  motion.spec.ts
│  ├─ comments.spec.ts  submit.spec.ts
│  └─ nojs.spec.ts  domains.spec.ts
└─ deploy/
   ├─ nginx/blog.conf
   ├─ systemd/{artalk,submitd}.service
   └─ scripts/00-baseline.sh
```

---

## 阶段一：工程地基

### Task 1: 仓库初始化（含 .gitignore 与构建脚本）

**Files:**
- Create: `.gitignore`, `package.json`, `astro.config.mjs`
- Test: `scripts/assert/build.mjs`

**Interfaces:**
- Produces: `pnpm build` / `pnpm compress` / `pnpm test:assert` 可用；`dist/index.html` 存在

- [ ] **Step 1: 写失败测试 `scripts/assert/build.mjs`**

```js
import { existsSync } from 'node:fs';
const p = 'dist/index.html';
if (!existsSync(p)) { console.error(`MISSING ${p}`); process.exit(1); }
console.log('BUILD OK');
```

- [ ] **Step 2: 运行确认失败**

Run: `node scripts/assert/build.mjs`
Expected: `MISSING dist/index.html`，退出码 1

- [ ] **Step 3: 写 `.gitignore`（v1 缺失，导致 `git add -A` 会提交 dist 与 node_modules）**

```gitignore
node_modules/
dist/
.astro/
*.br
*.gz
.env
.env.*
server/submitd/submitd
server/submitd/*.db
server/submitd/*.db-*
test-results/
playwright-report/
```

- [ ] **Step 4: 初始化并写 `package.json`（v1 声称 `pnpm build` 却从未定义）**

```bash
pnpm init
pnpm add astro@^7.4.1
pnpm add -D pagefind playwright @playwright/test
node -e "
const fs=require('fs');const p=JSON.parse(fs.readFileSync('package.json','utf8'));
p.type='module';
p.scripts={
  dev:'astro dev',
  build:'astro build && pagefind --site dist',
  compress:'node scripts/compress.mjs',
  'test:assert':'node scripts/assert/build.mjs && node scripts/assert/fonts.mjs',
  'test:e2e':'playwright test'
};
fs.writeFileSync('package.json',JSON.stringify(p,null,2));
"
```

- [ ] **Step 5: 写 `astro.config.mjs`（显式 assets 命名，供 immutable 缓存）**

```js
import { defineConfig } from 'astro/config';

export default defineConfig({
  site: 'https://blog.lancycelestia.com',
  build: {
    format: 'directory',
    assets: '_assets',
    inlineStylesheets: 'auto',
  },
  i18n: {
    locales: ['zh', 'en'],
    defaultLocale: 'zh',
    routing: { prefixDefaultLocale: false },
  },
});
```

- [ ] **Step 6: 写最小首页并构建**

`src/pages/index.astro`：
```astro
---
---
<!doctype html>
<html lang="zh"><head><meta charset="utf-8"><title>澜汐斋记</title></head>
<body><h1>澜汐斋记</h1></body></html>
```

Run: `pnpm build`
Expected: 退出码 0，`dist/index.html` 存在

- [ ] **Step 7: 运行测试确认通过**

Run: `pnpm test:assert`
Expected: `BUILD OK`（`fonts.mjs` 在 Task 2 建立，本步先只跑 build.mjs：`node scripts/assert/build.mjs`）

- [ ] **Step 8: Commit**

```bash
git init
git add -A
git commit -m "chore: init astro project with gitignore, build and assert scripts"
```

---

### Task 2: 设计 token 与字体策略

**Files:**
- Create: `src/styles/tokens.css`
- Test: `scripts/assert/fonts.mjs`, `scripts/assert/tokens.mjs`

**Interfaces:**
- Produces: `--glaze-*` `--font-display` `--font-sans` `--panel-*` `--dur-*` `--ease-glide`；三档 `--panel-blur` 精确值 0/14/24px

- [ ] **Step 1: 写失败测试 `scripts/assert/fonts.mjs`（**不用裸 grep**）**

```js
import { readdirSync, readFileSync, statSync } from 'node:fs';
import { join, extname } from 'node:path';

const BANNED = /\b(Inter|Roboto|Arial|Helvetica|Open Sans)\b/i;
const TEXT_EXT = new Set(['.css', '.html', '.js', '.mjs', '.json', '.svg', '.ts']);
const walk = (d) => readdirSync(d).flatMap((n) => {
  const p = join(d, n);
  return statSync(p).isDirectory() ? walk(p) : [p];
});

const files = walk('dist');
const hits = [];
for (const f of files) {
  if (!TEXT_EXT.has(extname(f))) continue;
  const txt = readFileSync(f, 'utf8');
  // 只在 font-family / @font-face src 的上下文里判定，避免 interface/transition 误命中
  for (const m of txt.matchAll(/font-family\s*:\s*([^;}]+)/gi)) {
    if (BANNED.test(m[1])) hits.push(`${f}: font-family → ${m[1].trim()}`);
  }
  for (const m of txt.matchAll(/@font-face\s*\{[^}]*src\s*:\s*url\(([^)]+)\)/gi)) {
    if (BANNED.test(m[1])) hits.push(`${f}: @font-face src → ${m[1]}`);
  }
}
// 反向断言：不得存在黑名单命名的字体文件
const badFiles = files.filter((f) => BANNED.test(f.split(/[\\/]/).pop()));
badFiles.forEach((f) => hits.push(`banned font file: ${f}`));

if (hits.length) { console.error('BANNED FONT:\n' + hits.join('\n')); process.exit(1); }
console.log('FONTS CLEAN');
```

- [ ] **Step 2: 运行确认失败**

Run: `node scripts/assert/fonts.mjs`
Expected: 因 `dist` 中尚无字体声明，可能直接 `FONTS CLEAN`——**这是允许的**（此时无字体）。真正的 RED 在 Step 6 的故意破坏步骤。

- [ ] **Step 3: 写失败测试 `scripts/assert/tokens.mjs`**

```js
import { readFileSync, readdirSync } from 'node:fs';
import { join } from 'node:path';

// 断言的是「编译产物里的 CSS」，不是源文件（T4）
const dir = 'dist/_assets';
let css = '';
try { css = readdirSync(dir).filter((f) => f.endsWith('.css'))
  .map((f) => readFileSync(join(dir, f), 'utf8')).join('\n'); } catch {}
if (!css) { console.error('NO COMPILED CSS — tokens.css likely not imported (T4 violation)'); process.exit(1); }

const must = ['--glaze-ink', '--glaze-abyss', '--glaze-celadon',
  '--glaze-coral', '--glaze-orchid', '--font-sans', '--ease-glide'];
const missing = must.filter((v) => !new RegExp(`${v}\\s*:\\s*[^;\\s]`).test(css));
if (missing.length) { console.error('MISSING TOKENS:', missing); process.exit(1); }

// 三档精确值（规格硬性数据）
const tiers = { low: '0px', mid: '14px', high: '24px' };
for (const [tier, val] of Object.entries(tiers)) {
  const re = new RegExp(`\\[data-motion=["']?${tier}["']?\\][^{]*\\{[^}]*--panel-blur\\s*:\\s*${val}`);
  if (!re.test(css)) { console.error(`TIER ${tier} WRONG (expect --panel-blur:${val})`); process.exit(1); }
}

// 正向断言字体：系统栈必须存在，不得为空
if (!/--font-sans\s*:\s*system-ui/.test(css)) { console.error('--font-sans must start with system-ui'); process.exit(1); }

console.log('TOKENS OK');
```

- [ ] **Step 4: 运行确认失败**

Run: `pnpm build && node scripts/assert/tokens.mjs`
Expected: `NO COMPILED CSS`，退出码 1

- [ ] **Step 5: 写 `tokens.css`**

```css
:root {
  color-scheme: light dark;

  --glaze-ink: #0B1622;
  --glaze-ink-soft: #41566B;
  --glaze-canvas: #F2F7FA;
  --glaze-porcelain: #FBFDFF;
  --glaze-abyss: #123A5C;
  --glaze-celadon: #3E9E8F;
  --glaze-coral: #F2729B;
  --glaze-orchid: #9B6BD6;
  --glaze-hairline: rgba(11,22,34,.08);
  --glaze-sheen: rgba(255,255,255,.5);
  --glaze-shadow: rgba(7,22,40,.45);

  --aurora-1: #3E9E8F;
  --aurora-2: #6FA8D8;
  --aurora-3: #9B6BD6;
  --aurora-4: #F2729B;
  --aurora-opacity: .4;

  --panel-bg: color-mix(in oklab, var(--glaze-abyss) 6%, var(--glaze-porcelain) 94%);
  --panel-sat: 150%;

  /* 字体：中文走系统栈，不加载 Web 字体 */
  --font-display: ui-serif, Georgia, "Songti SC", "Source Han Serif SC", serif;
  --font-sans: system-ui, -apple-system, "Segoe UI", "Microsoft YaHei", "PingFang SC", sans-serif;
  --font-mono: ui-monospace, "Cascadia Code", Consolas, monospace;

  --text-base: clamp(1rem, .95rem + .3vw, 1.0625rem);
  --text-h1: clamp(2.6rem, 6vw, 4.5rem);
  --text-h2: clamp(1.8rem, 3.4vw, 2.6rem);
  --leading-body: 1.75;
  --radius-lg: 2rem;
  --radius-inner: calc(var(--radius-lg) - 6px);

  --dur-fast: 240ms;
  --dur-base: 700ms;
  --ease-glide: cubic-bezier(.32,.72,0,1);
}

:root[data-theme="dark"] {
  --glaze-ink: #E8F1F8;
  --glaze-ink-soft: #9FB3C6;
  --glaze-canvas: #060B12;
  --glaze-porcelain: #0D151F;
  --glaze-abyss: #071628;
  --glaze-celadon: #5FD6C4;
  --glaze-coral: #FF8FB4;
  --glaze-orchid: #B58CF2;
  --glaze-hairline: rgba(232,241,248,.10);
  --glaze-sheen: rgba(255,255,255,.28);
  --glaze-shadow: rgba(0,0,0,.55);
  --aurora-1: #5FD6C4;
  --aurora-2: #4E8FD0;
  --aurora-3: #B58CF2;
  --aurora-4: #FF8FB4;
  --aurora-opacity: .8;
  --panel-bg: color-mix(in oklab, var(--glaze-abyss) 14%, var(--glaze-porcelain) 86%);
}

/* 三档：low 用 none 而非 blur(0px)，避免仍产生 backdrop 合成层 */
:root[data-motion="low"]  { --panel-blur: 0px;  --panel-sat: 100%; }
:root[data-motion="mid"]  { --panel-blur: 14px; --panel-sat: 150%; }
:root[data-motion="high"] { --panel-blur: 24px; --panel-sat: 180%; }
```

- [ ] **Step 6: 故意破坏验证测试会变红（T2 强制）**

临时把 `:root[data-motion="mid"]` 的 `14px` 改成 `15px`：
Run: `pnpm build && node scripts/assert/tokens.mjs`
Expected: `TIER mid WRONG`，退出码 1。**改回 `14px`。**

- [ ] **Step 7: 运行测试确认通过**

Run: `pnpm build && node scripts/assert/tokens.mjs && node scripts/assert/fonts.mjs`
Expected: `TOKENS OK` + `FONTS CLEAN`

- [ ] **Step 8: Commit**

```bash
git add src/styles/tokens.css scripts/assert/tokens.mjs scripts/assert/fonts.mjs
git commit -m "feat(design): tokens with motion tiers and system font stack"
```

---

## 阶段二：视觉层（四种风格）

### Task 3: 液态玻璃（仅浮层）+ Mica（滚动卡片）

**Files:**
- Create: `src/styles/glass.css`, `src/styles/mica.css`, `src/components/GlassPanel.astro`, `src/components/MicaCard.astro`
- Test: `scripts/assert/glass.mjs`

**Interfaces:**
- Consumes: `--panel-bg` `--panel-blur` `--panel-sat` `--glaze-sheen` `--glaze-shadow` `--radius-lg` `--radius-inner`
- Produces: `<GlassPanel className?>`（**用 className，与 astro-haze 一致**）、`<MicaCard className?>`

> **v1 教训**：v1 用 `class` 作 prop，与 Astro 保留属性冲突。`astro-haze` 原作者也用 `className` + `as?: keyof HTMLElementTagNameMap`，本版照此。

- [ ] **Step 1: 写失败测试 `scripts/assert/glass.mjs`（断言编译产物 + 组件导出）**

```js
import { readFileSync, readdirSync, existsSync } from 'node:fs';
import { join } from 'node:path';

const dir = 'dist/_assets';
const css = readdirSync(dir).filter((f) => f.endsWith('.css'))
  .map((f) => readFileSync(join(dir, f), 'utf8')).join('\n');

const fail = (m) => { console.error(m); process.exit(1); };

// 1) backdrop-filter 与 -webkit- 前缀必须成对，且标准属性在后
for (const m of css.matchAll(/([^{}]+)\{([^}]*backdrop-filter[^}]*)\}/g)) {
  const body = m[2];
  const hasWk = /-webkit-backdrop-filter\s*:/.test(body);
  const hasStd = /(?<!-webkit-)backdrop-filter\s*:/.test(body);
  if (hasStd && !hasWk) fail(`backdrop-filter without -webkit- prefix in: ${m[1].trim()}`);
}

// 2) Mica 必须含 feTurbulence 与 mix-blend-mode（v1 漏检的核心组成）
if (!/feTurbulence/.test(css)) fail('MISSING feTurbulence in compiled CSS — Mica noise not landed');
if (!/mix-blend-mode\s*:\s*overlay/.test(css)) fail('MISSING mix-blend-mode:overlay for Mica');

// 3) Mica 内所有 url() 只允许 data:image/svg+xml（禁止外部请求）
for (const m of css.matchAll(/url\((['"]?)([^)'"]+)\1\)/g)) {
  const v = m[2];
  if (v.startsWith('data:')) {
    if (!v.startsWith('data:image/svg+xml')) fail(`Mica uses non-svg data-URI: ${v.slice(0, 40)}`);
  } else if (!v.startsWith('/') && !v.startsWith('.')) {
    fail(`external url() found: ${v}`);
  }
}

// 4) 兜底：无 backdrop-filter 时面板必须有不透明 background（正向断言，不是查字符串存在）
if (!/@supports\s+not\s*\(\s*backdrop-filter/.test(css)) fail('NO @supports fallback');
const fbBlock = css.match(/@supports\s+not[^{]*\{([\s\S]*?)\n\}/);
if (!fbBlock || !/background\s*:\s*[^;]*(porcelain|#[0-9A-Fa-f]{3,8})/.test(fbBlock[1]))
  fail('FALLBACK BLOCK does not set an opaque background');

// 5) Double-Bezel：内壳必须存在
if (!/--radius-inner/.test(css)) fail('MISSING --radius-inner — Double-Bezel inner shell not implemented');

// 6) 组件文件存在且使用 className（v1 用 class 导致样式丢失）
for (const c of ['src/components/GlassPanel.astro', 'src/components/MicaCard.astro']) {
  if (!existsSync(c)) fail(`MISSING component ${c}`);
  const src = readFileSync(c, 'utf8');
  if (!/className/.test(src)) fail(`${c} must use className prop (v1 used class → styles lost)`);
}

console.log('GLASS OK');
```

- [ ] **Step 2: 运行确认失败**

Run: `node scripts/assert/glass.mjs`
Expected: FAIL（`dist/_assets` 无 CSS 或 MISSING component）

- [ ] **Step 3: 写 `glass.css`（只给浮层用）**

```css
/* 液态玻璃：仅用于 fixed/sticky 元素与小型浮层（Nav / 搜索弹层 / 设置面板）
   滚动文档流内的卡片请用 mica.css 的 .mica-card */
.glass {
  background: var(--panel-bg);
  -webkit-backdrop-filter: blur(var(--panel-blur)) saturate(var(--panel-sat));
  backdrop-filter: blur(var(--panel-blur)) saturate(var(--panel-sat));
  border: 1px solid var(--glaze-hairline);
  border-radius: var(--radius-lg);
  box-shadow: 0 24px 80px -32px var(--glaze-shadow), inset 0 1px 0 var(--glaze-sheen);
}

/* low 档：不留 backdrop 合成层 */
:root[data-motion="low"] .glass { -webkit-backdrop-filter: none; backdrop-filter: none; }

/* 无前缀支持的浏览器：不透明底色，保证可读 */
@supports not (backdrop-filter: blur(1px)) {
  .glass { background: var(--glaze-porcelain); }
}
```

- [ ] **Step 4: 写 `mica.css`（滚动卡片用，零 backdrop）**

```css
/* Mica 云母质感：用于滚动文档流内的卡片
   组成 = 取色底 + 内联噪点 + 漫射阴影。不使用 backdrop-filter。 */
.mica-card {
  position: relative;
  background: var(--panel-bg);
  border: 1px solid var(--glaze-hairline);
  border-radius: var(--radius-lg);
  box-shadow: 0 24px 80px -32px var(--glaze-shadow), inset 0 1px 0 var(--glaze-sheen);
  overflow: hidden;
}

/* 噪点：内联 SVG feTurbulence，零外部请求 */
.mica-card::before {
  content: '';
  position: absolute; inset: 0;
  pointer-events: none;
  opacity: .035;
  mix-blend-mode: overlay;
  background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='160' height='160'%3E%3Cfilter id='n'%3E%3CfeTurbulence type='fractalNoise' baseFrequency='0.8' numOctaves='3'/%3E%3C/filter%3E%3Crect width='160' height='160' filter='url(%23n)'/%3E%3C/svg%3E");
}

/* Double-Bezel 内壳 */
.mica-card__inner {
  position: relative;
  padding: 1.75rem;
  border-radius: var(--radius-inner);
  background: color-mix(in oklab, var(--glaze-porcelain) 70%, transparent);
  box-shadow: inset 0 1px 0 var(--glaze-sheen);
}
```

- [ ] **Step 5: 写两个组件**

`src/components/GlassPanel.astro`：
```astro
---
export interface Props {
  className?: string;
  as?: keyof HTMLElementTagNameMap;
}
const { className = '', as: Tag = 'div' } = Astro.props;
---
<Tag class={`glass ${className}`}><slot /></Tag>
```

`src/components/MicaCard.astro`：
```astro
---
export interface Props {
  className?: string;
  as?: keyof HTMLElementTagNameMap;
}
const { className = '', as: Tag = 'div' } = Astro.props;
---
<Tag class={`mica-card ${className}`}>
  <div class="mica-card__inner"><slot /></div>
</Tag>
```

- [ ] **Step 6: 在任务 5 的布局里 import 两个 CSS 后重跑（本步先验证样式能进编译产物）**

临时在 `src/pages/index.astro` 顶部加：
```astro
---
import '../styles/tokens.css';
import '../styles/glass.css';
import '../styles/mica.css';
---
```
Run: `pnpm build && node scripts/assert/glass.mjs`
Expected: `GLASS OK`

- [ ] **Step 7: 故意破坏验证（T2）**

删除 `mica.css` 里的 `feTurbulence` 那一行：
Run: `pnpm build && node scripts/assert/glass.mjs`
Expected: `MISSING feTurbulence in compiled CSS`，退出码 1。**改回。**

- [ ] **Step 8: Commit**

```bash
git add src/styles/glass.css src/styles/mica.css src/components/GlassPanel.astro src/components/MicaCard.astro scripts/assert/glass.mjs
git commit -m "feat(ui): liquid glass for overlays, mica for flow cards, double-bezel"
```

---

### Task 4: 动态釉瑚渐变（去 blur）+ 无闪烁主题 + 动效三档

**Files:**
- Create: `src/styles/global.css`, `src/components/AuroraBackground.astro`, `src/components/ThemeToggle.astro`, `src/components/MotionSwitch.astro`, `src/layouts/Base.astro`
- Test: `scripts/assert/motion.mjs`, `tests/e2e/motion.spec.ts`

**Interfaces:**
- Produces: `<Base title lang>`；`data-theme` `data-motion`；`<AuroraBackground />`

> **v1 教训 1**：Aurora 用 `filter: blur(80px)` + 3 层 70% 视口元素 + `will-change`，在低端安卓必然掉帧。**v2 用 `radial-gradient` 自身的柔化，去掉 `filter: blur()`。**
> **v1 教训 2**：`data-motion` 默认 `mid`，JS 未执行时动画照跑。**v2 默认 `low`，由 JS 升档。**

- [ ] **Step 1: 写失败测试 `scripts/assert/motion.mjs`（断言产物，不查源码字符串）**

```js
import { readFileSync, readdirSync } from 'node:fs';
import { join } from 'node:path';

const fail = (m) => { console.error(m); process.exit(1); };
const html = readFileSync('dist/index.html', 'utf8');
const css = readdirSync('dist/_assets').filter((f) => f.endsWith('.css'))
  .map((f) => readFileSync(join('dist/_assets', f), 'utf8')).join('\n');

// 1) 默认档必须是 low（无 JS 时静止）
const htmlTag = html.match(/<html[^>]*>/i)?.[0] ?? '';
if (!/data-motion="low"/.test(htmlTag))
  fail(`<html> must default to data-motion="low", got: ${htmlTag}`);

// 2) head 内必须有内联主题脚本，且在 </head> 之前
const head = html.slice(html.indexOf('<head'), html.indexOf('</head>'));
if (!/<script(?![^>]*\bsrc=)(?![^>]*type="module")[^>]*>[\s\S]*?localStorage[\s\S]*?<\/script>/i.test(head))
  fail('NO inline theme script in <head> — first-paint flash will occur (Review Focus 1)');
if (!/dataset\.theme/.test(head)) fail('inline script must set dataset.theme');

// 3) prefers-reduced-motion 必须在 CSS 层强制静止（不依赖 JS）
const rm = css.match(/@media\s*\(\s*prefers-reduced-motion\s*:\s*reduce\s*\)\s*\{([\s\S]*?)\}\s*\}/);
if (!rm) fail('NO @media (prefers-reduced-motion: reduce) in compiled CSS');
if (!/animation\s*:\s*none\s*!important/.test(rm[1]))
  fail('reduced-motion block must set animation:none !important');

// 4) @keyframes 里不得动画布局属性
for (const m of css.matchAll(/@keyframes\s+[\w-]+\s*\{([\s\S]*?)\n\}/g)) {
  const bad = m[1].match(/(?:^|[;{\s])(top|left|right|bottom|width|height|margin|padding)\s*:/);
  if (bad) fail(`@keyframes animates layout property "${bad[1]}"`);
}

// 5) Aurora 不得使用 filter: blur（性能）
const aur = readFileSync('src/components/AuroraBackground.astro', 'utf8');
if (/filter\s*:\s*blur\s*\(\s*[3-9]\dpx/.test(aur) || /filter\s*:\s*blur\s*\(\s*\d{3,}px/.test(aur))
  fail('Aurora must not use filter:blur(>30px) — use radial-gradient softness instead');

console.log('MOTION OK');
```

- [ ] **Step 2: 运行确认失败**

Run: `node scripts/assert/motion.mjs`
Expected: FAIL（`<html data-motion="low">` 不存在）

- [ ] **Step 3: 写 `global.css`（含 reduced-motion 与 focus）**

```css
*, *::before, *::after { box-sizing: border-box; }
html, body { margin: 0; }
body {
  font-family: var(--font-sans);
  font-size: var(--text-base);
  line-height: var(--leading-body);
  color: var(--glaze-ink);
  background: var(--glaze-canvas);
}
h1, h2 { font-family: var(--font-display); line-height: 1.1; letter-spacing: -.01em; }

/* 焦点可见（规格 §2.7） */
:focus-visible { outline: 2px solid var(--glaze-celadon); outline-offset: 3px; }
@media (forced-colors: active) { :focus-visible { outline-color: Highlight; } }

/* 动效降级：CSS 层强制，不依赖 JS（Review Focus 3） */
@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after {
    animation: none !important;
    transition-duration: .01ms !important;
    scroll-behavior: auto !important;
  }
}
```

- [ ] **Step 4: 写 `AuroraBackground.astro`（无 filter:blur）**

```astro
---
---
<div class="aurora" aria-hidden="true">
  <div class="aurora__b aurora__b--1"></div>
  <div class="aurora__b aurora__b--2"></div>
  <div class="aurora__b aurora__b--3"></div>
</div>

<style>
  /* 不用 filter: blur()：radial-gradient 的 color stop 本身即柔化，
     filter 会让被 transform 动画的大图层每帧重栅格化（v1 掉帧根因） */
  .aurora {
    position: fixed; inset: -10%; z-index: 0;
    pointer-events: none; overflow: hidden;
    opacity: var(--aurora-opacity);
  }
  .aurora__b {
    position: absolute; width: 80%; height: 80%; border-radius: 50%;
    will-change: transform;
  }
  .aurora__b--1 {
    background: radial-gradient(circle, var(--aurora-1) 0%, transparent 62%);
    top: -10%; left: -10%;
    animation: drift1 28s var(--ease-glide) infinite alternate;
  }
  .aurora__b--2 {
    background: radial-gradient(circle, var(--aurora-3) 0%, transparent 62%);
    bottom: -12%; right: -12%;
    animation: drift2 37s var(--ease-glide) infinite alternate;
  }
  .aurora__b--3 {
    background: radial-gradient(circle, var(--aurora-4) 0%, transparent 62%);
    top: 28%; left: 42%;
    animation: drift3 45s var(--ease-glide) infinite alternate;
  }
  @keyframes drift1 { from { transform: translate3d(0,0,0) scale(1); } to { transform: translate3d(6%,5%,0) scale(1.12); } }
  @keyframes drift2 { from { transform: translate3d(0,0,0) scale(1.08); } to { transform: translate3d(-8%,-6%,0) scale(.96); } }
  @keyframes drift3 { from { transform: translate3d(0,0,0) scale(.94); } to { transform: translate3d(-10%,6%,0) scale(1.14); } }

  /* 三档 */
  :root[data-motion="low"] .aurora__b { animation: none; will-change: auto; }
  :root[data-motion="mid"] .aurora__b--3 { animation: none; }
</style>
```

- [ ] **Step 5: 写 `Base.astro`（默认 low，内联脚本升档）**

```astro
---
import '../styles/tokens.css';
import '../styles/global.css';
import '../styles/glass.css';
import '../styles/mica.css';
import AuroraBackground from '../components/AuroraBackground.astro';

export interface Props { title: string; lang?: 'zh' | 'en' }
const { title, lang = 'zh' } = Astro.props;
---
<!doctype html>
<html lang={lang} data-theme="light" data-motion="low">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>{title}</title>
  <script is:inline>
    (function () {
      var r = document.documentElement;
      function safeGet(k) { try { return localStorage.getItem(k); } catch (e) { return null; } }
      function safeSet(k, v) { try { localStorage.setItem(k, v); } catch (e) {} }
      var reduce = matchMedia('(prefers-reduced-motion: reduce)').matches;
      var theme = safeGet('theme');
      if (theme) r.dataset.theme = theme;
      if (!reduce) {
        var m = safeGet('motion');
        if (m) { r.dataset.motion = m; }
        else {
          var weak = (navigator.hardwareConcurrency || 8) <= 4 || (navigator.deviceMemory || 8) <= 4;
          var noBlur = !(CSS.supports('backdrop-filter', 'blur(1px)') || CSS.supports('-webkit-backdrop-filter', 'blur(1px)'));
          r.dataset.motion = (weak || noBlur) ? 'mid' : 'high';
        }
      }
      window.__setTheme = function (v) { r.dataset.theme = v; safeSet('theme', v); };
      window.__setMotion = function (v) { r.dataset.motion = v; safeSet('motion', v); };
    })();
  </script>
</head>
<body>
  <AuroraBackground />
  <slot />
</body>
</html>
```

- [ ] **Step 6: 写切换组件**

`src/components/MotionSwitch.astro`：
```astro
---
---
<button id="motion-btn" type="button" aria-live="polite">动效：低</button>
<script>
  const ORDER = ['low', 'mid', 'high'] as const;
  const LABEL = { low: '低', mid: '中', high: '高' };
  const btn = document.getElementById('motion-btn')!;
  const root = document.documentElement;
  const sync = () => {
    const cur = (root.dataset.motion || 'low') as keyof typeof LABEL;
    btn.textContent = `动效：${LABEL[cur]}`;
  };
  btn.addEventListener('click', () => {
    const i = ORDER.indexOf((root.dataset.motion || 'low') as any);
    const next = ORDER[(i + 1) % ORDER.length];
    (window as any).__setMotion(next);
    sync();
  });
  sync();
</script>
```

`src/components/ThemeToggle.astro`：
```astro
---
---
<button id="theme-btn" type="button" aria-live="polite">明暗：浅釉</button>
<script>
  const btn = document.getElementById('theme-btn')!;
  const root = document.documentElement;
  const sync = () => {
    btn.textContent = `明暗：${root.dataset.theme === 'dark' ? '深釉' : '浅釉'}`;
  };
  btn.addEventListener('click', () => {
    (window as any).__setTheme(root.dataset.theme === 'dark' ? 'light' : 'dark');
    sync();
  });
  sync();
</script>
```

- [ ] **Step 7: 写 Playwright 行为测试 `tests/e2e/motion.spec.ts`**

```ts
import { test, expect } from '@playwright/test';

test('reduced-motion 下所有动画静止', async ({ page }) => {
  await page.emulateMedia({ reducedMotion: 'reduce' });
  await page.goto('/');
  const names = await page.$$eval('.aurora__b', (els) =>
    els.map((e) => getComputedStyle(e).animationName));
  expect(names.every((n) => n === 'none')).toBe(true);
});

test('high 档下 Aurora 有动画', async ({ page }) => {
  await page.addInitScript(() => localStorage.setItem('motion', 'high'));
  await page.goto('/');
  const name = await page.$eval('.aurora__b--1', (e) => getComputedStyle(e).animationName);
  expect(name).not.toBe('none');
});

test('localStorage 被禁用时不抛错且主题仍生效', async ({ page }) => {
  await page.addInitScript(() => {
    Object.defineProperty(window, 'localStorage', { get() { throw new Error('blocked'); } });
  });
  const errors: string[] = [];
  page.on('pageerror', (e) => errors.push(e.message));
  await page.goto('/');
  expect(errors).toHaveLength(0);
  await expect(page.locator('html')).toHaveAttribute('data-theme', 'light');
});
```

- [ ] **Step 8: 运行确认通过**

Run: `pnpm build && node scripts/assert/motion.mjs && pnpm exec playwright test tests/e2e/motion.spec.ts`
Expected: `MOTION OK` + 3 passed

- [ ] **Step 9: 故意破坏验证（T2）**

把 `Base.astro` 的 `data-motion="low"` 改成 `data-motion="mid"`：
Run: `pnpm build && node scripts/assert/motion.mjs`
Expected: `<html> must default to data-motion="low"`，退出码 1。**改回。**

- [ ] **Step 10: Commit**

```bash
git add src/styles/global.css src/components/AuroraBackground.astro src/components/ThemeToggle.astro src/components/MotionSwitch.astro src/layouts/Base.astro scripts/assert/motion.mjs tests/e2e/motion.spec.ts
git commit -m "feat(ui): aurora without blur filter, low-first motion tiers, a11y switches"
```

---

### Task 5: 内容集合 + 主页 Bento + 全套页面

**Files:**
- Create: `src/content.config.ts`, `src/content/blog/{hello.md,hello.en.md}`, `src/config/site.ts`, `src/i18n/ui.ts`
- Create: `src/pages/{index,about,links,404}.astro`, `src/pages/projects/index.astro`, `src/pages/blog/index.astro`, `src/pages/blog/[...slug].astro`, `src/pages/blog/page/[page].astro`, `src/pages/tags/index.astro`, `src/pages/tags/[tag].astro`
- Create: `src/components/{Nav,Footer}.astro`, `src/layouts/Base.astro`（Modify：接入 Nav/Footer）
- Test: `scripts/assert/content.mjs`, `tests/e2e/nojs.spec.ts`

**Interfaces:**
- Consumes: `<Base title lang>`、`<MicaCard className>`、`t(key, lang)`
- Produces: `getCollection('blog')` → `{ id, data: { title, lang, slug, date, tags, draft } }`

> **v1 教训**：`site.ts` 与 `about/links/projects` 页面在 v1 中只有声明、无任务创建 → 首页链到 404。本任务全部补齐。
> **v1 教训**：`p.id` 对 `hello.en.md` 会得到 `hello.en`，URL 带点且与 `/en/` 冲突。**v2 用 schema 的显式 `slug` 字段。**

- [ ] **Step 1: 写失败测试 `scripts/assert/content.mjs`**

```js
import { readFileSync, existsSync } from 'node:fs';
const fail = (m) => { console.error(m); process.exit(1); };

const idx = 'dist/index.html';
if (!existsSync(idx)) fail('MISSING dist/index.html');
const html = readFileSync(idx, 'utf8');

// 1) 正文必须出现在 HTML 里（Review Focus 5）
if (!/<h1[^>]*>[^<]*澜汐/.test(html)) fail('h1 missing or does not contain 澜汐');
if (!/<nav[\s>]/.test(html)) fail('NO <nav> in HTML — nav must not be JS-injected');
if (!/<footer[\s>]/.test(html)) fail('NO <footer> in HTML');

// 2) 草稿不得出现在主页
if (/DRAFT_SENTINEL/.test(html)) fail('draft post leaked to homepage');

// 3) 排序：新文章在前
const a = html.indexOf('开斋第一篇');
const b = html.indexOf('第二篇测试');
if (a === -1) fail('published post 开斋第一篇 not on homepage');
if (b !== -1 && b < a) fail('posts not sorted by date desc');

// 4) 首页所有内部链接的目标页必须真的存在（v1 首页链到 404）
const links = [...html.matchAll(/href="(\/[^"#?]*)"/g)].map((m) => m[1]);
const missing = [];
for (const l of new Set(links)) {
  if (l.startsWith('/_assets')) continue;
  const p = l.endsWith('/') ? `dist${l}index.html` : `dist${l}`;
  const alt = `dist${l.replace(/\/$/, '')}/index.html`;
  if (!existsSync(p) && !existsSync(alt)) missing.push(l);
}
if (missing.length) fail('BROKEN LINKS on homepage:\n' + missing.join('\n'));

console.log('CONTENT OK');
```

- [ ] **Step 2: 运行确认失败**

Run: `node scripts/assert/content.mjs`
Expected: FAIL（`h1 missing` 或 `BROKEN LINKS`）

- [ ] **Step 3: 写 `src/config/site.ts`（v1 声明却从未创建）**

```ts
export const site = {
  name: '澜汐斋记',
  displayName: '潮声茶馆',
  author: '澜汐',
  alias: '霞月',
  aliasEn: 'LancyCelestia',
  url: 'https://blog.lancycelestia.com',
  description: '潮声茶馆 · 澜汐的个人博客',
  icp: '', // 备案通过后填入，如「浙ICP备12345678号」
  features: {
    comments: 'artalk' as 'artalk' | 'off',
    submission: true,
    search: true,
  },
} as const;

export type Lang = 'zh' | 'en';
```

- [ ] **Step 4: 写 `src/i18n/ui.ts`（唯一文案源）**

```ts
export const ui = {
  zh: { latest: '最新文章', projects: '项目', about: '关于我', links: '友链',
        tags: '标签', search: '搜索', submit: '投稿', readMore: '阅读全文',
        archive: '归档', home: '首页', blog: '文章', noPosts: '还没有文章' },
  en: { latest: 'Latest posts', projects: 'Projects', about: 'About',
        links: 'Links', tags: 'Tags', search: 'Search', submit: 'Submit',
        readMore: 'Read more', archive: 'Archive', home: 'Home', blog: 'Blog',
        noPosts: 'No posts yet' },
} as const;

export type Lang = keyof typeof ui;
export const t = (k: keyof typeof ui.zh, lang: Lang): string => ui[lang][k];
```

- [ ] **Step 5: 写 `content.config.ts`（含显式 slug）**

```ts
import { defineCollection, z } from 'astro:content';
import { glob } from 'astro/loaders';

const blog = defineCollection({
  loader: glob({ pattern: '**/*.md', base: './src/content/blog' }),
  schema: z.object({
    title: z.string(),
    lang: z.enum(['zh', 'en']).default('zh'),
    slug: z.string(),
    date: z.coerce.date(),
    tags: z.array(z.string()).default([]),
    draft: z.boolean().default(false),
  }),
});
export const collections = { blog };
```

- [ ] **Step 6: 写示例文章（含草稿）**

`src/content/blog/hello.md`：
```markdown
---
title: 开斋第一篇
lang: zh
slug: hello
date: 2026-10-20
tags: [随记]
---
潮声起于岸，文字落于纸。
```

`src/content/blog/second.md`：
```markdown
---
title: 第二篇测试
lang: zh
slug: second
date: 2026-10-21
tags: [测试]
---
用于验证排序。
```

`src/content/blog/draft.md`：
```markdown
---
title: DRAFT_SENTINEL
lang: zh
slug: draft
date: 2026-10-22
draft: true
---
不应出现在任何列表里。
```

`src/content/blog/hello.en.md`：
```markdown
---
title: The first entry
lang: en
slug: hello
date: 2026-10-20
tags: [notes]
---
Tide rises on the shore.
```

- [ ] **Step 7: 写 `Nav.astro` 与 `Footer.astro`**

`src/components/Nav.astro`：浮动胶囊，使用 `.glass`（**fixed，符合 backdrop-filter 约束**）
```astro
---
import { t } from '../i18n/ui';
import type { Lang } from '../i18n/ui';
import MotionSwitch from './MotionSwitch.astro';
import ThemeToggle from './ThemeToggle.astro';
interface Props { lang?: Lang }
const { lang = 'zh' } = Astro.props;
const base = lang === 'zh' ? '' : '/en';
---
<nav class="glass nav">
  <a href={`${base}/`} class="nav__brand">潮声茶馆</a>
  <a href={`${base}/blog/`}>{t('blog', lang)}</a>
  <a href={`${base}/tags/`}>{t('tags', lang)}</a>
  <a href={`${base}/projects/`}>{t('projects', lang)}</a>
  <a href={`${base}/about/`}>{t('about', lang)}</a>
  <a href={`${base}/links/`}>{t('links', lang)}</a>
  <MotionSwitch />
  <ThemeToggle />
</nav>
<style>
  .nav {
    position: fixed; top: 1.5rem; left: 50%; transform: translateX(-50%);
    z-index: 100; display: flex; gap: 1rem; align-items: center;
    padding: .625rem 1.25rem; border-radius: 9999px; width: max-content;
    max-width: calc(100vw - 2rem); flex-wrap: wrap;
  }
  .nav a { color: var(--glaze-ink); text-decoration: none; font-size: .9rem; }
  .nav__brand { font-family: var(--font-display); font-size: 1rem; }
  @media (max-width: 767.98px) { .nav { top: .75rem; gap: .625rem; padding: .5rem .875rem; } }
</style>
```

`src/components/Footer.astro`：含 ICP 悬挂
```astro
---
import { site } from '../config/site';
---
<footer class="footer">
  <p>© {new Date().getFullYear()} {site.author} · {site.displayName}</p>
  {site.icp && (
    <p>
      <a href="https://beian.miit.gov.cn/" target="_blank" rel="noopener noreferrer">{site.icp}</a>
    </p>
  )}
</footer>
<style>
  .footer { padding: 4rem 1rem 3rem; text-align: center; color: var(--glaze-ink-soft); font-size: .875rem; }
</style>
```

- [ ] **Step 8: 修改 `Base.astro` 接入 Nav/Footer**

```astro
---
import '../styles/tokens.css';
import '../styles/global.css';
import '../styles/glass.css';
import '../styles/mica.css';
import AuroraBackground from '../components/AuroraBackground.astro';
import Nav from '../components/Nav.astro';
import Footer from '../components/Footer.astro';

export interface Props { title: string; lang?: 'zh' | 'en' }
const { title, lang = 'zh' } = Astro.props;
---
<!doctype html>
<html lang={lang} data-theme="light" data-motion="low">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>{title}</title>
  <link rel="canonical" href="https://blog.lancycelestia.com/" />
  <script is:inline>/* …与 Task 4 Step 5 完全相同，勿改动… */</script>
</head>
<body>
  <AuroraBackground />
  <Nav lang={lang} />
  <main><slot /></main>
  <Footer />
</body>
</html>
```

- [ ] **Step 9: 写主页与全部页面**

`src/pages/index.astro`（Bento，卡片用 **MicaCard 而非 GlassPanel**）：
```astro
---
import Base from '../layouts/Base.astro';
import MicaCard from '../components/MicaCard.astro';
import { getCollection } from 'astro:content';
import { t } from '../i18n/ui';

const posts = (await getCollection('blog', ({ data }) => data.lang === 'zh' && !data.draft))
  .sort((a, b) => b.data.date.valueOf() - a.data.date.valueOf())
  .slice(0, 5);
---
<Base title="澜汐斋记 · 潮声茶馆">
  <div class="bento">
    <section class="hero">
      <p class="eyebrow">潮声茶馆</p>
      <h1>澜汐</h1>
      <p class="sub">霞月 · LancyCelestia</p>
    </section>

    <MicaCard className="cell span-6">
      <h2>{t('latest', 'zh')}</h2>
      <ul>{posts.map((p) => (
        <li><a href={`/blog/${p.data.slug}/`}>{p.data.title}</a></li>
      ))}</ul>
    </MicaCard>

    <MicaCard className="cell span-3">
      <h2>{t('projects', 'zh')}</h2>
      <p><a href="/projects/">chatbot · QQ 聊天机器人</a></p>
    </MicaCard>

    <MicaCard className="cell span-3">
      <h2>{t('about', 'zh')}</h2>
      <p><a href="/about/">澜汐</a></p>
    </MicaCard>

    <MicaCard className="cell span-12">
      <h2>{t('archive', 'zh')}</h2>
      <p><a href="/blog/">全部文章</a> · <a href="/tags/">标签云</a></p>
    </MicaCard>
  </div>
</Base>

<style>
  .bento {
    display: grid; grid-template-columns: repeat(12, 1fr);
    gap: 1.5rem; max-width: 1200px; margin-inline: auto;
    padding: clamp(6rem, 12vw, 9rem) clamp(1rem, 4vw, 4rem) clamp(2rem, 6vw, 5rem);
  }
  .hero { grid-column: span 12; min-height: 38dvh; display: grid; align-content: center; }
  .eyebrow { font-size: .75rem; letter-spacing: .2em; text-transform: uppercase; color: var(--glaze-ink-soft); }
  h1 { font-size: var(--text-h1); margin: .5rem 0; }
  .sub { color: var(--glaze-ink-soft); }
  .span-6 { grid-column: span 6; }
  .span-3 { grid-column: span 3; }
  .span-12 { grid-column: span 12; }
  @media (max-width: 767.98px) {
    .bento { grid-template-columns: 1fr; padding-inline: 1rem; }
    .hero, .span-6, .span-3, .span-12 { grid-column: span 1; }
    .hero { min-height: auto; }
  }
</style>
```

其余页面（about / links / projects / blog index / tags / 404）**均用 `<Base>` 包裹**，`about.astro` 写个人简介 + 项目简介（chatbot QQ 机器人），`links.astro` 先放空态文案。**不得裸写 `<html>`**（v1 的 `submit.astro` 就犯了这个错）。

`src/pages/blog/[...slug].astro` 用 `getStaticPaths` 从 `slug` 派生：
```astro
---
export async function getStaticPaths() {
  const { getCollection } = await import('astro:content');
  const posts = await getCollection('blog', ({ data }) => data.lang === 'zh' && !data.draft);
  return posts.map((p) => ({ params: { slug: p.data.slug }, props: { entry: p } }));
}
const { entry } = Astro.props;
const { Content } = await entry.render();
---
```

- [ ] **Step 10: 写 Playwright 无 JS 测试 `tests/e2e/nojs.spec.ts`**

```ts
import { test, expect } from '@playwright/test';

test.use({ javaScriptEnabled: false });

test('无 JS 时正文与导航仍在', async ({ page }) => {
  await page.goto('/');
  await expect(page.locator('h1')).toContainText('澜汐');
  await expect(page.locator('nav')).toBeVisible();
  await expect(page.locator('footer')).toBeVisible();
});
```

- [ ] **Step 11: 运行确认通过**

Run: `pnpm build && node scripts/assert/content.mjs && pnpm exec playwright test tests/e2e/nojs.spec.ts`
Expected: `CONTENT OK` + 1 passed

- [ ] **Step 12: 故意破坏验证（T2）**

删除 `draft: true` 的过滤条件（改为不过滤）：
Run: `pnpm build && node scripts/assert/content.mjs`
Expected: `draft post leaked to homepage`，退出码 1。**改回。**

- [ ] **Step 13: Commit**

```bash
git add src/content.config.ts src/content src/config src/i18n src/components/Nav.astro src/components/Footer.astro src/layouts/Base.astro src/pages scripts/assert/content.mjs tests/e2e/nojs.spec.ts
git commit -m "feat(content): collections with slug, bento homepage, all pages, no-404 links"
```

---

### Task 6: 中英双语路由 + hreflang + canonical

**Files:**
- Create: `src/pages/en/index.astro`, `src/pages/en/blog/[...slug].astro`, `src/pages/en/blog/index.astro`, `src/components/LangSwitch.astro`
- Modify: `src/layouts/Base.astro`（Props 增加 `hreflang` 输出）
- Test: `scripts/assert/i18n.mjs`

**Interfaces:**
- Consumes: `t(key, lang)`、`site.url`
- Produces: `/en/*` 镜像；`<base>` 输出三条 `alternate` link

- [ ] **Step 1: 写失败测试 `scripts/assert/i18n.mjs`**

```js
import { readFileSync, existsSync } from 'node:fs';
const fail = (m) => { console.error(m); process.exit(1); };

for (const p of ['dist/en/index.html', 'dist/index.html']) {
  if (!existsSync(p)) fail(`MISSING ${p}`);
}
const en = readFileSync('dist/en/index.html', 'utf8');
const zh = readFileSync('dist/index.html', 'utf8');

// 1) lang 属性正确
if (!/<html[^>]*lang="en"/.test(en)) fail('EN page lang wrong');
if (!/<html[^>]*lang="zh"/.test(zh)) fail('ZH page lang wrong');

// 2) 三条 hreflang 齐全且 href 精确（v1 只查 hreflang 存在）
const expect = (html, page) => {
  const pairs = [['zh','https://blog.lancycelestia.com/'],
                 ['en','https://blog.lancycelestia.com/en/'],
                 ['x-default','https://blog.lancycelestia.com/']];
  for (const [lang, href] of pairs) {
    const re = new RegExp(`<link[^>]+rel="alternate"[^>]*hreflang="${lang}"[^>]*href="${href}"`);
    const re2 = new RegExp(`<link[^>]+href="${href}"[^>]*hreflang="${lang}"`);
    if (!re.test(html) && !re2.test(html)) fail(`${page}: missing hreflang="${lang}" href="${href}"`);
  }
};
expect(en, 'EN');
expect(zh, 'ZH');

// 3) 英文页 UI 不得含中文（扫 i18n 字典，而非单串黑名单）
const i18nSrc = readFileSync('src/i18n/ui.ts', 'utf8');
const enBlock = i18nSrc.match(/en:\s*\{([\s\S]*?)\}/)?.[1] ?? '';
if (/[\u4e00-\u9fff]/.test(enBlock)) fail('en dictionary contains Chinese characters');

// 4) canonical 必须指向 blog 主域
if (!/<link[^>]+rel="canonical"[^>]+href="https:\/\/blog\.lancycelestia\.com\//.test(zh))
  fail('missing canonical to blog.lancycelestia.com');

console.log('I18N OK');
```

- [ ] **Step 2: 运行确认失败**

Run: `node scripts/assert/i18n.mjs`
Expected: FAIL（`MISSING dist/en/index.html`）

- [ ] **Step 3: 在 `Base.astro` 输出 hreflang 与 canonical**

把 `<head>` 内的 `link` 替换为：
```astro
  <link rel="canonical" href="https://blog.lancycelestia.com/" />
  <link rel="alternate" hreflang="zh" href="https://blog.lancycelestia.com/" />
  <link rel="alternate" hreflang="en" href="https://blog.lancycelestia.com/en/" />
  <link rel="alternate" hreflang="x-default" href="https://blog.lancycelestia.com/" />
```

- [ ] **Step 4: 写 `src/pages/en/index.astro`（全部文案走 `t(_, 'en')`）**

结构与 `src/pages/index.astro` 相同，但 `lang="en"`、`t(k,'en')`，文章取 `data.lang === 'en'`，链接前缀 `/en`。

- [ ] **Step 5: 写 `src/pages/en/blog/[...slug].astro` 与 `en/blog/index.astro`**

英文 `getStaticPaths` 只取 `data.lang === 'en'`，路径 `/en/blog/<slug>/`。

- [ ] **Step 6: 写 `LangSwitch.astro`（在 Nav 右侧）**

```astro
---
interface Props { current: 'zh' | 'en' }
const { current } = Astro.props;
const other = current === 'zh' ? 'en' : 'zh';
const href = other === 'zh' ? '/' : '/en/';
---
<a href={href} class="lang" hreflang={other} lang={other}>{other === 'en' ? 'EN' : '中文'}</a>
```

- [ ] **Step 7: 运行确认通过**

Run: `pnpm build && node scripts/assert/i18n.mjs`
Expected: `I18N OK`

- [ ] **Step 8: 故意破坏验证（T2）**

删掉 `Base.astro` 里 `hreflang="x-default"` 那一行：
Run: `pnpm build && node scripts/assert/i18n.mjs`
Expected: `missing hreflang="x-default"`，退出码 1。**改回。**

- [ ] **Step 9: Commit**

```bash
git add src/pages/en src/components/LangSwitch.astro src/layouts/Base.astro scripts/assert/i18n.mjs
git commit -m "feat(i18n): english mirror with canonical and accurate hreflang"
```

---

### Task 7: 站内搜索（Pagefind，按语言分面）

**Files:**
- Create: `src/components/SearchModal.astro`
- Modify: `src/components/Nav.astro`（加搜索按钮）、`package.json`（build 已含 pagefind）
- Test: `scripts/assert/search.mjs`, `tests/e2e/search.spec.ts`

- [ ] **Step 1: 写失败测试 `scripts/assert/search.mjs`**

```js
import { existsSync, readdirSync, statSync } from 'node:fs';
const fail = (m) => { console.error(m); process.exit(1); };

if (!existsSync('dist/pagefind/pagefind-ui.js')) fail('MISSING pagefind-ui.js — index not built');
const frag = 'dist/pagefind/fragment';
if (!existsSync(frag)) fail('MISSING pagefind/fragment dir');
const n = readdirSync(frag).length;
if (n === 0) fail('pagefind fragment dir is empty — index has no content');

// 首页必须带 lang，Pagefind 才能做语言分面
const home = readdirSync('dist/index.html') && 'dist/index.html';
if (!/lang="zh"/.test(require('node:fs').readFileSync(home, 'utf8'))) fail('no lang attr');

console.log(`SEARCH OK (${n} fragments)`);
```

- [ ] **Step 2: 运行确认失败**

Run: `node scripts/assert/search.mjs`
Expected: `MISSING pagefind-ui.js`

- [ ] **Step 3: 写 `SearchModal.astro`（⌘K 打开，含语言过滤）**

```astro
---
import type { Lang } from '../i18n/ui';
interface Props { lang?: Lang }
const { lang = 'zh' } = Astro.props;
---
<button id="search-open" class="glass search-btn" type="button" aria-label="搜索">⌘K</button>
<dialog id="search-dialog" class="glass search-dialog">
  <div id="search-root"></div>
  <button id="search-close" type="button">关闭</button>
</dialog>

<script define:vars={{ lang }}>
  const dlg = document.getElementById('search-dialog');
  const root = document.getElementById('search-root');
  let mounted = false;

  async function open() {
    if (!mounted) {
      await import('/pagefind/pagefind-ui.js');
      new window.PagefindUI({ element: root, showSubResults: true, showImages: false });
      mounted = true;
    }
    dlg.showModal();
  }
  document.getElementById('search-open').addEventListener('click', open);
  document.getElementById('search-close').addEventListener('click', () => dlg.close());
  document.addEventListener('keydown', (e) => {
    if ((e.metaKey || e.ctrlKey) && e.key === 'k') { e.preventDefault(); open(); }
  });
</script>
```

> Pagefind 的 `pagefind --site dist` 会自动按 `<html lang>` 建立语言分面；`showSubResults` 提升中文检索召回。若需强制按当前页语言过滤，在 `PagefindUI` 的 `filters` 里加 `{ 'lang': lang }`——**Task 7 之后单独验证**。

- [ ] **Step 4: 写 Playwright 搜索测试**

```ts
import { test, expect } from '@playwright/test';

test('搜索能命中文章正文独有词', async ({ page }) => {
  await page.goto('/');
  await page.click('#search-open');
  await page.fill('.pagefind-ui__search-input', '潮声起于岸');
  await expect(page.locator('.pagefind-ui__result')).not.toHaveCount(0);
});
```

- [ ] **Step 5: 运行确认通过**

Run: `pnpm build && node scripts/assert/search.mjs && pnpm exec playwright test tests/e2e/search.spec.ts`
Expected: `SEARCH OK` + 1 passed

- [ ] **Step 6: Commit**

```bash
git add src/components/SearchModal.astro src/components/Nav.astro scripts/assert/search.mjs tests/e2e/search.spec.ts
git commit -m "feat(search): pagefind modal with language-aware index"
```

---

### Task 8: RSS / Sitemap / 资源约束

**Files:**
- Create: `src/pages/rss.xml.ts`
- Modify: `astro.config.mjs`（加 sitemap 集成）
- Test: `scripts/assert/assets.mjs`

- [ ] **Step 1: 写失败测试 `scripts/assert/assets.mjs`**

```js
import { readdirSync, statSync, existsSync } from 'node:fs';
import { join, extname } from 'node:path';
const fail = (m) => { console.error(m); process.exit(1); };

if (!existsSync('dist/rss.xml')) fail('MISSING rss.xml');
if (!existsSync('dist/sitemap-index.xml')) fail('MISSING sitemap (astro sitemap integration not enabled)');

const walk = (d) => readdirSync(d).flatMap((n) => {
  const p = join(d, n);
  return statSync(p).isDirectory() ? walk(p) : [p];
});
const files = walk('dist');

// 许可约束：不得打包 WE 工坊资源（v1 只查 .pkg，漏了贴图/音频）
const FORBIDDEN = ['.pkg', '.tex', '.webm', '.mp3', '.wav'];
const bad = files.filter((f) => FORBIDDEN.includes(extname(f).toLowerCase()));
if (bad.length) fail('WE workshop assets packaged:\n' + bad.join('\n'));

// 备案号：占位符不得上线（v1 允许 XXXXXXXX 通过）
for (const f of files.filter((x) => x.endsWith('.html'))) {
  const t = require('node:fs').readFileSync(f, 'utf8');
  if (/XXXX/.test(t)) fail(`${f} contains placeholder ICP number`);
}

console.log('ASSETS OK');
```

- [ ] **Step 2: 运行确认失败**

Run: `node scripts/assert/assets.mjs`
Expected: `MISSING rss.xml`

- [ ] **Step 3: 写 `rss.xml.ts` 并在 `astro.config.mjs` 启用 sitemap**

```bash
pnpm add @astrojs/sitemap @astrojs/rss
```

`astro.config.mjs` 加：
```js
import sitemap from '@astrojs/sitemap';
// 在 defineConfig 中：
integrations: [sitemap()],
```

`src/pages/rss.xml.ts` 用 `@astrojs/rss` 生成中文文章 feed。

- [ ] **Step 4: 运行确认通过**

Run: `pnpm build && node scripts/assert/assets.mjs`
Expected: `ASSETS OK`

- [ ] **Step 5: Commit**

```bash
git add src/pages/rss.xml.ts astro.config.mjs scripts/assert/assets.mjs
git commit -m "feat: rss, sitemap, and asset policy assertions"
```

---

### Task 9: 潮线签名元素（含豁免声明）

**Files:**
- Create: `src/components/TideLine.astro`
- Modify: `src/pages/index.astro`（章节间插入）
- Test: `tests/e2e/tideline.spec.ts`

> **约束豁免（显式记录）**：Global Constraints #9 要求只用 `transform`/`opacity`。潮线按规格 §2.6 需要 `stroke-dasharray` 动画，它**不是** transform/opacity。**本任务改用 `stroke-dashoffset`**（可动画的呈现属性，走 SVG 重绘而非布局），并在规格 §2.6 同步改为「stroke-dashoffset」。这是**唯一**的豁免，已在此记录。

- [ ] **Step 1: 写失败测试 `tests/e2e/tideline.spec.ts`**

```ts
import { test, expect } from '@playwright/test';

test('潮线存在且 reduced-motion 下静止', async ({ page }) => {
  await page.emulateMedia({ reducedMotion: 'reduce' });
  await page.goto('/');
  const path = page.locator('.tideline path');
  await expect(path).toHaveCount(1);
  const name = await path.evaluate((e) => getComputedStyle(e).animationName);
  expect(name).toBe('none');
});

test('潮线在 high 档有动画', async ({ page }) => {
  await page.addInitScript(() => localStorage.setItem('motion', 'high'));
  await page.goto('/');
  const name = await page.locator('.tideline path').evaluate((e) => getComputedStyle(e).animationName);
  expect(name).not.toBe('none');
});
```

- [ ] **Step 2: 运行确认失败**

Run: `pnpm exec playwright test tests/e2e/tideline.spec.ts`
Expected: FAIL（`.tideline path` 数量为 0）

- [ ] **Step 3: 写 `TideLine.astro`**

```astro
---
---
<div class="tideline" aria-hidden="true">
  <svg viewBox="0 0 1200 40" preserveAspectRatio="none" xmlns="http://www.w3.org/2000/svg">
    <defs>
      <linearGradient id="tide-grad" x1="0" y1="0" x2="1" y2="0">
        <stop offset="0%" stop-color="var(--aurora-1)" />
        <stop offset="33%" stop-color="var(--aurora-2)" />
        <stop offset="66%" stop-color="var(--aurora-3)" />
        <stop offset="100%" stop-color="var(--aurora-4)" />
      </linearGradient>
    </defs>
    <path d="M0,20 C150,4 300,36 450,20 C600,4 750,36 900,20 C1050,4 1150,30 1200,20"
          pathLength="1" stroke="url(#tide-grad)" stroke-width="1.5"
          fill="none" stroke-dasharray="1" />
  </svg>
</div>
<style>
  .tideline { width: 100%; max-width: 1200px; margin: 4rem auto; opacity: .8; }
  .tideline svg { width: 100%; height: 40px; display: block; }
  .tideline path {
    stroke-dashoffset: 1;
    animation: tide-sweep 8s var(--ease-glide) infinite;
  }
  @keyframes tide-sweep {
    from { stroke-dashoffset: 1; }
    to   { stroke-dashoffset: -1; }
  }
  :root[data-motion="low"] .tideline path { animation: none; stroke-dashoffset: 0; }
</style>
```

- [ ] **Step 4: 运行确认通过**

Run: `pnpm build && pnpm exec playwright test tests/e2e/tideline.spec.ts`
Expected: 2 passed

- [ ] **Step 5: Commit**

```bash
git add src/components/TideLine.astro src/pages/index.astro tests/e2e/tideline.spec.ts
git commit -m "feat(ui): tide line signature with stroke-dashoffset exemption"
```

---


---

### Task 10: 参考质感移植（胶片颗粒 + 滚动进场 + Hero 重塑）

**Files:**
- Create: `src/components/Grain.astro`, `src/components/Reveal.astro`, `src/scripts/reveal.ts`
- Modify: `src/pages/index.astro`（Hero 结构改造）、`src/layouts/Base.astro`（挂载 Grain）
- Test: `scripts/assert/reference.mjs`, `tests/e2e/reveal.spec.ts`

**Interfaces:**
- Consumes: `data-motion`（三档）、`--glaze-*` token
- Produces: `<Grain />`、`<Reveal delay?>`、`.is-revealed` 类

**依据**：`docs/superpowers/specs/2026-10-20-reference-daniya-analysis.md`，用户决策 **B = 1 + 3**。

> **只移植两样东西**：① 全局胶片颗粒（daniya 的 `grain-shift`）；② 滚动进场（daniya 的 `panel-in` + IntersectionObserver）。
> **色板不换**、不加泡泡、不加圆环、不加暖橙。潮线继续承担视觉锚点。

- [ ] **Step 1: 写失败测试 `scripts/assert/reference.mjs`**

```js
import { readFileSync, readdirSync } from 'node:fs';
import { join } from 'node:path';
const fail = (m) => { console.error(m); process.exit(1); };

const css = readdirSync('dist/_assets').filter((f) => f.endsWith('.css'))
  .map((f) => readFileSync(join('dist/_assets', f), 'utf8')).join('\n');
const html = readFileSync('dist/index.html', 'utf8');

// 1) 颗粒层必须存在于 DOM 且用内联 SVG 噪点（零外部请求）
if (!/class="[^"]*\bgrain\b/.test(html)) fail('MISSING .grain layer in HTML');
if (!/grain/.test(css)) fail('MISSING .grain CSS');
const grainBlock = css.match(/\.grain[^{]*\{([^}]*)\}/);
if (!grainBlock) fail('cannot locate .grain rule');
if (!/feTurbulence/.test(grainBlock[1])) fail('.grain must use inline SVG feTurbulence');
if (!/position\s*:\s*fixed/.test(grainBlock[1])) fail('.grain must be position:fixed');
if (!/pointer-events\s*:\s*none/.test(grainBlock[1])) fail('.grain must not block pointer events');

// 2) 颗粒动画必须只位移、且用 steps 制造跳帧质感（daniya 用 steps(6)）
const kf = css.match(/@keyframes\s+grain-shift\s*\{([\s\S]*?)\n\}/);
if (!kf) fail('MISSING @keyframes grain-shift');
if (/(top|left|width|height)\s*:/.test(kf[1])) fail('grain-shift must not animate layout props');
if (!/animation\s*:[^;]*steps\(/.test(grainBlock[1])) fail('.grain should use steps() for film-grain feel');

// 3) 滚动进场：必须有 .is-revealed 的最终态，且初始态是隐藏
if (!/\.is-revealed/.test(css)) fail('MISSING .is-revealed rule');
const revInit = css.match(/\.reveal[^{]*\{([^}]*)\}/);
if (!revInit) fail('MISSING .reveal initial rule');
if (!/opacity\s*:\s*0/.test(revInit[1])) fail('.reveal initial state must be opacity:0');
if (!/translate[Yy]\(/.test(revInit[1]) && !/translateY/.test(css)) fail('.reveal must translate before reveal');

// 4) reduced-motion 必须让 reveal 立即可见、颗粒静止（CSS 层强制）
const rm = css.match(/@media\s*\(\s*prefers-reduced-motion\s*:\s*reduce\s*\)\s*\{([\s\S]*?)\n\}/);
if (!rm) fail('NO reduced-motion block');
if (!/\.reveal/.test(rm[1])) fail('reduced-motion must force .reveal visible');
if (!/\.grain/.test(rm[1])) fail('reduced-motion must stop .grain animation');

// 5) 主色不得被替换成 daniya 的暖橙
if (/#ffb98a/i.test(css)) fail('must NOT adopt daniya ember color #ffb98a');

console.log('REFERENCE OK');
```

- [ ] **Step 2: 运行确认失败**

Run: `node scripts/assert/reference.mjs`
Expected: `MISSING .grain layer in HTML`，退出码 1

- [ ] **Step 3: 写 `src/components/Grain.astro`**

```astro
---
---
<div class="grain" aria-hidden="true"></div>

<style>
  /* 移植自 daniya.top 的 .grain：内联 SVG feTurbulence，零图片请求
     差异：9s steps(6) 跳帧位移 → 胶片颗粒的"沙沙"感 */
  .grain {
    position: fixed;
    inset: -50%;
    z-index: 60;
    pointer-events: none;
    opacity: .05;
    background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='180' height='180'%3E%3Cfilter id='n'%3E%3CfeTurbulence type='fractalNoise' baseFrequency='.9' numOctaves='2' stitchTiles='stitch'/%3E%3C/filter%3E%3Crect width='180' height='180' filter='url(%23n)'/%3E%3C/svg%3E");
    animation: grain-shift 9s steps(6) infinite;
  }

  @keyframes grain-shift {
    0%, 100% { transform: translate(0, 0); }
    20%      { transform: translate(-3%, 2%); }
    40%      { transform: translate(2%, -3%); }
    60%      { transform: translate(-2%, -2%); }
    80%      { transform: translate(3%, 1%); }
  }

  /* mid：保留噪点但不跳帧（省 GPU） */
  :root[data-motion="mid"] .grain { animation: none; }
  /* low：整个颗粒层不渲染 */
  :root[data-motion="low"] .grain { display: none; }

  /* reduced-motion：CSS 层强制静止（不依赖 JS 的 data-motion） */
  @media (prefers-reduced-motion: reduce) {
    .grain { animation: none; display: none; }
  }
</style>
```

- [ ] **Step 4: 写 `src/scripts/reveal.ts`（IntersectionObserver）**

```ts
// 移植自 daniya.top 的 panel-in 滚动进场（它用了 17 处 IntersectionObserver）
// 差异：只做 fade + translateY，不做 blur（blur 会整块重绘）

const REDUCED = () => window.matchMedia('(prefers-reduced-motion: reduce)').matches;
const MOTION = () => document.documentElement.dataset.motion ?? 'low';

export function initReveal(): void {
  const els = Array.from(document.querySelectorAll<HTMLElement>('.reveal'));

  // low 档或 reduced-motion：不做动画，直接标记可见
  if (REDUCED() || MOTION() === 'low') {
    els.forEach((el) => el.classList.add('is-revealed'));
    return;
  }

  if (!('IntersectionObserver' in window)) {
    els.forEach((el) => el.classList.add('is-revealed'));
    return;
  }

  const io = new IntersectionObserver(
    (entries) => {
      for (const entry of entries) {
        if (!entry.isIntersecting) continue;
        const el = entry.target as HTMLElement;
        const delay = Number(el.dataset.revealDelay ?? 0);
        window.setTimeout(() => el.classList.add('is-revealed'), delay);
        io.unobserve(el);
      }
    },
    { rootMargin: '0px 0px -10% 0px', threshold: 0.12 },
  );

  els.forEach((el) => io.observe(el));
}

if (typeof document !== 'undefined') {
  if (document.readyState === 'loading') {
    document.addEventListener('DOMContentLoaded', initReveal, { once: true });
  } else {
    initReveal();
  }
}
```

- [ ] **Step 5: 写 `src/components/Reveal.astro`**

```astro
---
export interface Props {
  className?: string;
  delay?: number;
  as?: keyof HTMLElementTagNameMap;
}
const { className = '', delay = 0, as: Tag = 'div' } = Astro.props;
---
<Tag class={`reveal ${className}`} data-reveal-delay={delay}>
  <slot />
</Tag>

<style>
  .reveal {
    opacity: 0;
    transform: translateY(16px);
    transition:
      opacity 800ms var(--ease-glide),
      transform 800ms var(--ease-glide);
    will-change: opacity, transform;
  }
  .reveal.is-revealed {
    opacity: 1;
    transform: translateY(0);
    will-change: auto;
  }

  /* mid 档缩短时长 */
  :root[data-motion="mid"] .reveal {
    transition-duration: 400ms;
  }

  /* low 或 reduced-motion：直接可见，无动画（CSS 层强制，不依赖 JS） */
  :root[data-motion="low"] .reveal {
    opacity: 1;
    transform: none;
    transition: none;
  }
  @media (prefers-reduced-motion: reduce) {
    .reveal {
      opacity: 1;
      transform: none;
      transition: none;
    }
  }
</style>
```

- [ ] **Step 6: 在 `Base.astro` 挂载 Grain**

```astro
---
import '../styles/tokens.css';
import '../styles/global.css';
import '../styles/glass.css';
import '../styles/mica.css';
import AuroraBackground from '../components/AuroraBackground.astro';
import Grain from '../components/Grain.astro';
import Nav from '../components/Nav.astro';
import Footer from '../components/Footer.astro';
// ...
---
<body>
  <AuroraBackground />
  <Grain />
  <Nav lang={lang} />
  <main><slot /></main>
  <Footer />
</body>
```

- [ ] **Step 7: 改造主页 Hero（学 daniya 的展现方式）**

`src/pages/index.astro` 的 `<div class="bento">` 之前插入：

```astro
---
import Reveal from '../components/Reveal.astro';
import TideLine from '../components/TideLine.astro';
---
<section class="hero" aria-labelledby="hero-title">
  <!-- 中心发光：对应 daniya 的中心圆环，由 CSS radial-gradient 实现，零图片 -->
  <div class="hero__halo" aria-hidden="true"></div>

  <Reveal>
    <p class="hero__eyebrow">WUTHERING · 潮声茶馆 · 澜汐斋记</p>
    <h1 id="hero-title" class="hero__title">澜汐</h1>
    <p class="hero__sub">霞月 · LancyCelestia</p>
    <p class="hero__lede">把潮声收进一盏茶，然后轻轻地，看过它的梦。</p>
    <div class="hero__tags">
      <span>个人博客</span><span>技术随记</span><span>chatbot</span>
    </div>
  </Reveal>

  <TideLine />
</section>

<style>
  .hero {
    position: relative;
    min-height: 82dvh;
    display: grid;
    place-items: center;
    text-align: center;
    padding-inline: 1rem;
    overflow: hidden;
  }

  /* 中心发光：只有中心亮，边缘纯黑（daniya 质感法则 3） */
  .hero__halo {
    position: absolute;
    inset: 0;
    pointer-events: none;
    background: radial-gradient(
      ellipse 46% 52% at 50% 46%,
      color-mix(in oklab, var(--glaze-celadon) 18%, transparent) 0%,
      color-mix(in oklab, var(--glaze-orchid) 10%, transparent) 38%,
      transparent 72%
    );
  }

  /* 极细字重靠体量压场（daniya 质感法则 2） */
  .hero__title {
    font-size: clamp(4rem, 13vw, 11rem);
    font-weight: 300;
    letter-spacing: .06em;
    line-height: .95;
    margin: 0;
    background: linear-gradient(100deg,
      var(--glaze-celadon) 0%,
      var(--glaze-abyss) 34%,
      var(--glaze-orchid) 68%,
      var(--glaze-coral) 100%);
    -webkit-background-clip: text;
    background-clip: text;
    color: transparent;
  }

  /* 极小字号辅助信息（daniya 质感法则 6） */
  .hero__eyebrow {
    font-size: .6875rem;
    letter-spacing: .34em;
    text-transform: uppercase;
    color: var(--glaze-ink-soft);
    margin: 0 0 1.5rem;
  }
  .hero__sub {
    font-size: .875rem;
    letter-spacing: .18em;
    color: var(--glaze-ink-soft);
    margin: 1.25rem 0 0;
  }
  .hero__lede {
    max-width: 26em;
    margin: 2rem auto 0;
    color: var(--glaze-ink);
    line-height: 1.85; /* daniya 行高 */
    font-size: 1rem;
  }
  .hero__tags {
    display: flex;
    gap: .5rem;
    justify-content: center;
    flex-wrap: wrap;
    margin-top: 2rem;
  }
  .hero__tags span {
    font-size: .75rem;
    padding: .375rem .875rem;
    border: 1px solid var(--glaze-hairline);
    border-radius: 9999px;
    color: var(--glaze-ink-soft);
  }

  @media (max-width: 767.98px) {
    .hero { min-height: auto; padding-block: 5rem; }
    .hero__title { font-size: clamp(3rem, 16vw, 5rem); }
  }
</style>
```

> 同时把主页 Bento 的每张 `<MicaCard>` 包进 `<Reveal delay={i * 80}>`，让卡片逐张淡入。

- [ ] **Step 8: 写 Playwright 行为测试 `tests/e2e/reveal.spec.ts`**

```ts
import { test, expect } from '@playwright/test';

test('滚动进场：卡片初始隐藏，滚入后显示', async ({ page }) => {
  await page.addInitScript(() => localStorage.setItem('motion', 'high'));
  await page.goto('/');
  const first = page.locator('.reveal').first();
  // 滚到可见区域
  await first.scrollIntoViewIfNeeded();
  await expect(first).toHaveClass(/is-revealed/, { timeout: 3000 });
  expect(await first.evaluate((e) => getComputedStyle(e).opacity)).toBe('1');
});

test('reduced-motion 下卡片直接可见，无过渡', async ({ page }) => {
  await page.emulateMedia({ reducedMotion: 'reduce' });
  await page.goto('/');
  const first = page.locator('.reveal').first();
  expect(await first.evaluate((e) => getComputedStyle(e).opacity)).toBe('1');
  expect(await first.evaluate((e) => getComputedStyle(e).transitionDuration)).toBe('0s');
});

test('low 档颗粒层不渲染', async ({ page }) => {
  await page.emulateMedia({ reducedMotion: 'reduce' });
  await page.goto('/');
  const display = await page.locator('.grain').evaluate((e) => getComputedStyle(e).display);
  expect(display).toBe('none');
});

test('high 档颗粒层存在且有 steps 动画', async ({ page }) => {
  await page.addInitScript(() => localStorage.setItem('motion', 'high'));
  await page.goto('/');
  const g = page.locator('.grain');
  await expect(g).toBeVisible();
  const name = await g.evaluate((e) => getComputedStyle(e).animationName);
  expect(name).toBe('grain-shift');
  const timing = await g.evaluate((e) => getComputedStyle(e).animationTimingFunction);
  expect(timing).toContain('steps');
});

test('未采用 daniya 的暖橙', async ({ page }) => {
  await page.goto('/');
  const titleStyle = await page.locator('.hero__title').evaluate((e) => getComputedStyle(e).backgroundImage);
  expect(titleStyle.toLowerCase()).not.toContain('255, 185, 138'); // #ffb98a
});
```

- [ ] **Step 9: 运行确认通过**

Run: `pnpm build && node scripts/assert/reference.mjs && pnpm exec playwright test tests/e2e/reveal.spec.ts`
Expected: `REFERENCE OK` + 5 passed

- [ ] **Step 10: 故意破坏验证（T2）**

把 `Grain.astro` 的 `animation: grain-shift 9s steps(6) infinite;` 改成 `none`：
Run: `pnpm build && node scripts/assert/reference.mjs`
Expected: `.grain should use steps()`，退出码 1。**改回。**

- [ ] **Step 11: Commit**

```bash
git add src/components/Grain.astro src/components/Reveal.astro src/scripts/reveal.ts src/layouts/Base.astro src/pages/index.astro scripts/assert/reference.mjs tests/e2e/reveal.spec.ts
git commit -m "feat(visual): port daniya grain texture, scroll reveal and hero structure"
```

---

## 阶段三：构建优化与壁纸

### Task 11: 文本白名单预压（修正 v1 的「压一切」错误）

**Files:**
- Create: `scripts/compress.mjs`, `scripts/assert/compress.mjs`
- Modify: `package.json`

> **v1 教训**：v1 对 `dist` 全部文件做 Brotli q11，包括已压缩的 AVIF/WOFF2/Pagefind 索引 → `.br` 反而**变大**，而 Nginx 会优先发它，**流量反而增加**。B 审查明确点名此条。

- [ ] **Step 1: 写失败测试 `scripts/assert/compress.mjs`**

```js
import { readdirSync, statSync, existsSync } from 'node:fs';
import { join, extname } from 'node:path';
const fail = (m) => { console.error(m); process.exit(1); };

const TEXT = new Set(['.html', '.css', '.js', '.mjs', '.svg', '.xml', '.json', '.txt', '.webmanifest']);
const SKIP = new Set(['.avif', '.webp', '.woff2', '.png', '.jpg', '.jpeg', '.gif', '.ico', '.pf_fragment', '.pf_meta']);

const walk = (d) => readdirSync(d).flatMap((n) => {
  const p = join(d, n);
  if (/.+\.(br|gz)$/.test(n)) return [];
  return statSync(p).isDirectory() ? walk(p) : [p];
});

const files = walk('dist');
let checked = 0, brTotal = 0, origTotal = 0;
for (const f of files) {
  const ext = extname(f).toLowerCase();
  if (SKIP.has(ext)) {
    // 反向断言：已压缩格式不得生成 .br
    if (existsSync(f + '.br')) fail(`SHOULD NOT precompress binary: ${f}.br`);
    continue;
  }
  if (!TEXT.has(ext)) continue;
  if (!existsSync(f + '.br')) fail(`MISSING ${f}.br`);
  if (!existsSync(f + '.gz')) fail(`MISSING ${f}.gz`);
  const o = statSync(f).size;
  const b = statSync(f + '.br').size;
  if (b >= o) fail(`.br is NOT smaller: ${f} (${o} → ${b})`);
  checked++; brTotal += b; origTotal += o;
}

// 首屏预算（全局断言，v1 缺失）
const idx = 'dist/index.html';
if (existsSync(idx)) {
  const iBr = statSync(idx + '.br').size;
  if (iBr > 150 * 1024) fail(`index.html.br exceeds 150KB budget: ${iBr} bytes`);
}

console.log(`COMPRESS OK (${checked} files, ${origTotal} → ${brTotal} bytes, ${(brTotal/origTotal*100).toFixed(1)}%)`);
```

- [ ] **Step 2: 运行确认失败**

Run: `node scripts/assert/compress.mjs`
Expected: `MISSING dist/index.html.br`

- [ ] **Step 3: 写 `scripts/compress.mjs`（白名单）**

```js
import { readdirSync, statSync, readFileSync, writeFileSync } from 'node:fs';
import { join, extname } from 'node:path';
import { brotliCompressSync, gzipSync, constants } from 'node:zlib';

const TEXT = new Set(['.html', '.css', '.js', '.mjs', '.svg', '.xml', '.json', '.txt', '.webmanifest']);
const MIN_SIZE = 256; // 太小的文件压了反而更大

const walk = (d) => readdirSync(d).flatMap((n) => {
  const p = join(d, n);
  return statSync(p).isDirectory() ? walk(p) : [p];
});

let n = 0;
for (const f of walk('dist')) {
  if (!TEXT.has(extname(f).toLowerCase())) continue;
  const size = statSync(f).size;
  if (size < MIN_SIZE) continue;
  const buf = readFileSync(f);
  const br = brotliCompressSync(buf, {
    params: {
      [constants.BROTLI_PARAM_QUALITY]: 11,
      [constants.BROTLI_PARAM_SIZE_HINT]: buf.length,
    },
  });
  if (br.length < buf.length) { writeFileSync(f + '.br', br); n++; }
  const gz = gzipSync(buf, { level: 9 });
  if (gz.length < buf.length) writeFileSync(f + '.gz', gz);
}
console.log(`compressed ${n} files`);
```

- [ ] **Step 4: 运行确认通过**

Run: `pnpm build && pnpm compress && node scripts/assert/compress.mjs`
Expected: `COMPRESS OK (…)`

- [ ] **Step 5: 故意破坏验证（T2）**

把 `TEXT` 里加入 `'.avif'`：
Run: `pnpm build && pnpm compress && node scripts/assert/compress.mjs`
Expected: `SHOULD NOT precompress binary`（若有 avif 文件）或覆盖率断言失败。**改回。**

- [ ] **Step 6: Commit**

```bash
git add scripts/compress.mjs scripts/assert/compress.mjs package.json
git commit -m "feat(build): brotli/gzip precompression limited to text whitelist"
```

---

### Task 12: 壁纸入口（CSS 默认 + webwallgl 按需）

**Files:**
- Create: `src/components/Wallpaper.astro`
- Modify: `src/layouts/Base.astro`（可选挂载）
- Test: `tests/e2e/wallpaper.spec.ts`

**决策依据**：用户选 **B3/C**（两者都做）。`webwallgl` 未压缩 **2.66 MiB**，按 0.8 元/GB ≈ **0.0021 元/次**。必须在按钮上明示。

- [ ] **Step 1: 写失败测试 `tests/e2e/wallpaper.spec.ts`**

```ts
import { test, expect } from '@playwright/test';

test('默认不加载 webwallgl（零额外流量）', async ({ page }) => {
  const requests: string[] = [];
  page.on('request', (r) => requests.push(r.url()));
  await page.goto('/');
  await page.waitForTimeout(1500);
  expect(requests.some((u) => u.includes('webwallgl'))).toBe(false);
});

test('按钮标明体积与费用', async ({ page }) => {
  await page.goto('/');
  const btn = page.locator('#wallpaper-enable');
  await expect(btn).toContainText('2.7 MB');
  await expect(btn).toContainText('流量');
});

test('low 档下按钮禁用', async ({ page }) => {
  await page.emulateMedia({ reducedMotion: 'reduce' });
  await page.goto('/');
  await expect(page.locator('#wallpaper-enable')).toBeDisabled();
});
```

- [ ] **Step 2: 运行确认失败**

Run: `pnpm exec playwright test tests/e2e/wallpaper.spec.ts`
Expected: FAIL（`#wallpaper-enable` 不存在）

- [ ] **Step 3: 写 `Wallpaper.astro`**

```astro
---
---
<section class="wallpaper-setup">
  <h2>动态壁纸</h2>
  <p>默认使用 CSS 釉瑚渐变（零流量）。启用 Wallpaper Engine 壁纸需下载约 <strong>2.7 MB</strong> 引擎，按流量计费约 <strong>0.002 元/次</strong>。</p>
  <button id="wallpaper-enable" type="button">启用 WE 壁纸（2.7 MB · 产生流量）</button>
  <p id="wallpaper-status" role="status"></p>
</section>

<script>
  const btn = document.getElementById('wallpaper-enable') as HTMLButtonElement;
  const status = document.getElementById('wallpaper-status')!;
  const root = document.documentElement;

  const syncDisabled = () => {
    btn.disabled = root.dataset.motion === 'low';
    btn.title = btn.disabled ? '低动效档位下不启用壁纸' : '';
  };
  syncDisabled();
  new MutationObserver(syncDisabled).observe(root, { attributes: true, attributeFilter: ['data-motion'] });

  btn.addEventListener('click', async () => {
    btn.disabled = true;
    status.textContent = '正在加载引擎…';
    try {
      // ponytail: 2.66MiB 引擎按需加载；若将来体积失控，改为只保留 CSS 渐变
      const mod: any = await import('webwallgl');
      const canvas = document.createElement('canvas');
      canvas.className = 'wallpaper-canvas';
      document.body.prepend(canvas);
      await mod.mount(canvas, { source: '/wallpapers/default' });
      status.textContent = '壁纸已启用。';
    } catch (e) {
      status.textContent = '引擎加载失败，已保持 CSS 渐变背景。';
      btn.disabled = false;
    }
  });
</script>
```

> `webwallgl` 的真实 API 需在 P3 阶段实测（B 审查指出它的 `main` 指向 UMD 构建）。**若 `import()` 的 API 与 README 不符，按 `ponytail:` 注释降级为仅 CSS 渐变，并在此任务内记录实测结果。**

- [ ] **Step 4: 运行确认通过**

Run: `pnpm build && pnpm exec playwright test tests/e2e/wallpaper.spec.ts`
Expected: 3 passed

- [ ] **Step 5: Commit**

```bash
git add src/components/Wallpaper.astro tests/e2e/wallpaper.spec.ts
git commit -m "feat(wallpaper): opt-in webwallgl with explicit size and cost"
```

---

## 阶段四：后端（Artalk + 自研投稿）

### Task 13: 自研投稿服务 `submitd`（Go + SQLite）

**Files:**
- Create: `server/submitd/{go.mod,main.go,store.go,handlers.go,store_test.go,handlers_test.go}`
- Test: `cd server/submitd && go test ./...`

**Interfaces:**
- Produces: `POST /posts` 创建投稿；`GET /posts` 列出（需管理员）；`GET /healthz`
- 端口 `23367`

> **架构决策**：Artalk **没有**投稿能力（B 审查确认其 README 与 HTTP API 均无 posts/投稿实体）。用户选 **2B 自研**。本服务独立于 Artalk，内存约 20–30 MB。

- [ ] **Step 1: 写失败测试 `store_test.go`**

```go
package main

import "testing"

func TestCreateAndList(t *testing.T) {
	s, err := OpenStore(":memory:")
	if err != nil { t.Fatal(err) }
	defer s.Close()

	id, err := s.CreatePost("标题", "正文内容", "澜汐")
	if err != nil { t.Fatal(err) }
	if id == 0 { t.Fatal("expected non-zero id") }

	posts, err := s.ListPosts()
	if err != nil { t.Fatal(err) }
	if len(posts) != 1 { t.Fatalf("want 1 post, got %d", len(posts)) }
	if posts[0].Title != "标题" { t.Fatalf("title mismatch: %s", posts[0].Title) }
}

func TestRejectEmpty(t *testing.T) {
	s, _ := OpenStore(":memory:")
	defer s.Close()
	if _, err := s.CreatePost("", "x", "a"); err == nil {
		t.Fatal("expected error for empty title")
	}
}
```

- [ ] **Step 2: 运行确认失败**

Run: `cd server/submitd && go mod init submitd && go test ./...`
Expected: 编译失败（`OpenStore` 未定义）

- [ ] **Step 3: 写 `store.go`**

```go
package main

import (
	"database/sql"
	"errors"
	"strings"
	"time"

	_ "modernc.org/sqlite"
)

type Post struct {
	ID        int64
	Title     string
	Body      string
	Author    string
	CreatedAt time.Time
}

type Store struct{ db *sql.DB }

func OpenStore(dsn string) (*Store, error) {
	db, err := sql.Open("sqlite", dsn)
	if err != nil { return nil, err }
	_, err = db.Exec(`CREATE TABLE IF NOT EXISTS posts (
		id INTEGER PRIMARY KEY AUTOINCREMENT,
		title TEXT NOT NULL,
		body TEXT NOT NULL,
		author TEXT NOT NULL,
		created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
	)`)
	if err != nil { db.Close(); return nil, err }
	return &Store{db: db}, nil
}

func (s *Store) Close() error { return s.db.Close() }

func (s *Store) CreatePost(title, body, author string) (int64, error) {
	if strings.TrimSpace(title) == "" { return 0, errors.New("title required") }
	if strings.TrimSpace(body) == "" { return 0, errors.New("body required") }
	res, err := s.db.Exec(`INSERT INTO posts (title, body, author) VALUES (?, ?, ?)`, title, body, author)
	if err != nil { return 0, err }
	return res.LastInsertId()
}

func (s *Store) ListPosts() ([]Post, error) {
	rows, err := s.db.Query(`SELECT id, title, body, author, created_at FROM posts ORDER BY created_at DESC`)
	if err != nil { return nil, err }
	defer rows.Close()
	var out []Post
	for rows.Next() {
		var p Post
		if err := rows.Scan(&p.ID, &p.Title, &p.Body, &p.Author, &p.CreatedAt); err != nil {
			return nil, err
		}
		out = append(out, p)
	}
	return out, rows.Err()
}
```

- [ ] **Step 4: 运行测试确认通过**

Run: `cd server/submitd && go get modernc.org/sqlite && go test ./...`
Expected: `ok submitd`

- [ ] **Step 5: 写失败测试 `handlers_test.go`**

```go
package main

import (
	"net/http/httptest"
	"strings"
	"testing"
)

func TestHealthz(t *testing.T) {
	mux := newMux(mustStore(t))
	req := httptest.NewRequest("GET", "/healthz", nil)
	w := httptest.NewRecorder()
	mux.ServeHTTP(w, req)
	if w.Code != 200 { t.Fatalf("want 200, got %d", w.Code) }
}

func TestPostRequiresAuth(t *testing.T) {
	mux := newMux(mustStore(t))
	req := httptest.NewRequest("POST", "/posts", strings.NewReader(`{"title":"t","body":"b"}`))
	w := httptest.NewRecorder()
	mux.ServeHTTP(w, req)
	if w.Code != 401 { t.Fatalf("want 401 without token, got %d", w.Code) }
}

func mustStore(t *testing.T) *Store {
	s, err := OpenStore(":memory:")
	if err != nil { t.Fatal(err) }
	t.Cleanup(func() { s.Close() })
	return s
}
```

- [ ] **Step 6: 运行确认失败**

Run: `cd server/submitd && go test ./...`
Expected: `undefined: newMux`

- [ ] **Step 7: 写 `handlers.go` 与 `main.go`**

```go
// handlers.go
package main

import (
	"crypto/subtle"
	"encoding/json"
	"net/http"
	"os"
	"strings"
)

func newMux(s *Store) *http.ServeMux {
	mux := http.NewServeMux()

	mux.HandleFunc("GET /healthz", func(w http.ResponseWriter, r *http.Request) {
		w.WriteHeader(http.StatusOK)
		w.Write([]byte(`{"ok":true}`))
	})

	mux.HandleFunc("POST /posts", func(w http.ResponseWriter, r *http.Request) {
		if !authed(r) { http.Error(w, "unauthorized", http.StatusUnauthorized); return }
		var in struct{ Title, Body, Author string }
		if err := json.NewDecoder(r.Body).Decode(&in); err != nil {
			http.Error(w, "bad json", http.StatusBadRequest); return
		}
		id, err := s.CreatePost(in.Title, in.Body, in.Author)
		if err != nil { http.Error(w, err.Error(), http.StatusBadRequest); return }
		w.Header().Set("Content-Type", "application/json")
		json.NewEncoder(w).Encode(map[string]any{"id": id})
	})

	mux.HandleFunc("GET /posts", func(w http.ResponseWriter, r *http.Request) {
		if !authed(r) { http.Error(w, "unauthorized", http.StatusUnauthorized); return }
		posts, err := s.ListPosts()
		if err != nil { http.Error(w, "db error", http.StatusInternalServerError); return }
		w.Header().Set("Content-Type", "application/json")
		json.NewEncoder(w).Encode(posts)
	})

	return mux
}

func authed(r *http.Request) bool {
	token := os.Getenv("SUBMITD_TOKEN")
	if token == "" { return false }
	got := strings.TrimPrefix(r.Header.Get("Authorization"), "Bearer ")
	return subtle.ConstantTimeCompare([]byte(got), []byte(token)) == 1
}
```

```go
// main.go
package main

import (
	"log"
	"net/http"
	"os"
	"time"
)

func main() {
	dsn := os.Getenv("SUBMITD_DSN")
	if dsn == "" { dsn = "/var/lib/submitd/submitd.db" }
	s, err := OpenStore(dsn)
	if err != nil { log.Fatal(err) }
	defer s.Close()

	srv := &http.Server{
		Addr:              ":23367",
		Handler:           newMux(s),
		ReadHeaderTimeout: 5 * time.Second,
		ReadTimeout:       15 * time.Second,
		WriteTimeout:      15 * time.Second,
	}
	log.Println("submitd listening on :23367")
	log.Fatal(srv.ListenAndServe())
}
```

- [ ] **Step 8: 运行测试确认通过**

Run: `cd server/submitd && go test ./... && go build -o submitd .`
Expected: `ok submitd`，生成 `submitd` 可执行文件

- [ ] **Step 9: Commit**

```bash
git add server/submitd
git commit -m "feat(submitd): go + sqlite submission service with auth and tests"
```

---

### Task 14: 评论组件（Artalk，正确的 API 路径）

**Files:**
- Create: `src/components/Comments.astro`
- Modify: `src/pages/blog/[...slug].astro`（插入 `<Comments />`）
- Test: `tests/e2e/comments.spec.ts`

> **v1 致命错误（B 审查）**：v1 用 `/api/` 探测健康 + `/api/users/me` 判登录，但 Artalk 的 API 在 **`/api/v2/`**，且**没有** `/users/me`。加上反代 `proxy_pass …/`（尾斜杠）会裁剪路径 → 必然 404 → **永远显示「评论暂不可用」**，bug 被降级逻辑掩盖。
>
> **v2**：反代**不带尾斜杠**（保留 `/api/v2` 原样），健康判据改为 `GET /api/v2/` 且**要求 JSON 合法**。

- [ ] **Step 1: 写失败测试 `tests/e2e/comments.spec.ts`**

```ts
import { test, expect } from '@playwright/test';

test('后端正常时评论区可渲染', async ({ page }) => {
  await page.route('**/api/v2/**', (r) =>
    r.fulfill({ status: 200, contentType: 'application/json', body: '{"comments":[]}' }));
  await page.goto('/blog/hello/');
  await expect(page.locator('#comments-slot')).not.toContainText('评论暂不可用');
});

test('后端 500 时降级文案出现且正文仍在', async ({ page }) => {
  await page.route('**/api/v2/**', (r) => r.fulfill({ status: 500, body: 'boom' }));
  await page.goto('/blog/hello/');
  await expect(page.locator('h1')).toBeVisible();
  await expect(page.locator('#comments-slot')).toContainText('评论暂不可用');
});

test('后端超时时降级', async ({ page }) => {
  await page.route('**/api/v2/**', (r) => r.abort());
  await page.goto('/blog/hello/');
  await expect(page.locator('#comments-slot')).toContainText('评论暂不可用');
});
```

- [ ] **Step 2: 运行确认失败**

Run: `pnpm exec playwright test tests/e2e/comments.spec.ts`
Expected: FAIL（`#comments-slot` 不存在）

- [ ] **Step 3: 写 `Comments.astro`**

```astro
---
import { site } from '../config/site';
const enabled = site.features.comments === 'artalk';
---
{enabled && (
  <section class="comments" data-pagefind-ignore>
    <h2>评论</h2>
    <div id="comments-slot"><p class="muted">正在加载评论…</p></div>
  </section>
)}

<script>
  const slot = document.getElementById('comments-slot');
  if (slot) {
    const ctrl = new AbortController();
    const timer = setTimeout(() => ctrl.abort(), 4000);

    // 健康判据：必须是合法 JSON（v1 只判非 5xx，导致 404 也被当成功）
    fetch('/api/v2/', { signal: ctrl.signal, headers: { Accept: 'application/json' } })
      .then(async (r) => {
        if (!r.ok) throw new Error(`HTTP ${r.status}`);
        await r.json(); // 不是 JSON 就抛错
        slot.innerHTML = '<div id="artalk-root"></div>';
      })
      .catch(() => {
        slot.innerHTML = '<p class="muted">评论暂不可用，稍后刷新试试。</p>';
      })
      .finally(() => clearTimeout(timer));
  }
</script>
```

- [ ] **Step 4: 运行确认通过**

Run: `pnpm build && pnpm exec playwright test tests/e2e/comments.spec.ts`
Expected: 3 passed

- [ ] **Step 5: Commit**

```bash
git add src/components/Comments.astro src/pages/blog/[...slug].astro tests/e2e/comments.spec.ts
git commit -m "feat(comments): artalk with correct /api/v2 path and json health check"
```

---

### Task 15: 投稿页面（走 Base，不裸写 html）

**Files:**
- Create: `src/pages/submit.astro`
- Test: `scripts/assert/submit.mjs`, `tests/e2e/submit.spec.ts`

> **v1 错误**：v1 的 `submit.astro` 裸写 `<html lang="zh">`，绕过 `Base` → **没有 tokens/glass/主题**，暗色模式点进去变浅色（C 审查 P1-17）。

- [ ] **Step 1: 写失败测试 `scripts/assert/submit.mjs`**

```js
import { readFileSync } from 'node:fs';
const fail = (m) => { console.error(m); process.exit(1); };
const src = readFileSync('src/pages/submit.astro', 'utf8');
if (!/import\s+Base\s+from/.test(src)) fail('submit.astro must import Base (v1 wrote a bare <html>)');
if (/<html[\s>]/.test(src)) fail('submit.astro must NOT contain a bare <html> tag');
console.log('SUBMIT OK');
```

- [ ] **Step 2: 运行确认失败**

Run: `node scripts/assert/submit.mjs`
Expected: `ENOENT` 或 `must import Base`

- [ ] **Step 3: 写 `submit.astro`（用 Base + MicaCard）**

```astro
---
import Base from '../layouts/Base.astro';
import MicaCard from '../components/MicaCard.astro';
import { site } from '../config/site';
---
<Base title="投稿 · 澜汐斋记">
  <div class="wrap">
    <MicaCard>
      {site.features.submission ? (
        <>
          <h1>投稿</h1>
          <form id="submit-form" novalidate>
            <label>标题 <input name="title" required maxlength="120" /></label>
            <label>署名 <input name="author" required maxlength="40" /></label>
            <label>正文 <textarea name="body" required rows="12"></textarea></label>
            <button type="submit">提交审阅</button>
            <p id="submit-status" role="status"></p>
          </form>
        </>
      ) : (
        <p>投稿功能暂未开放。</p>
      )}
    </MicaCard>
  </div>
</Base>

<style>
  .wrap { max-width: 42rem; margin: 8rem auto 4rem; padding-inline: 1rem; }
  label { display: block; margin-block: 1rem; }
  input, textarea { width: 100%; font: inherit; padding: .5rem; }
</style>

<script>
  const form = document.getElementById('submit-form') as HTMLFormElement | null;
  const status = document.getElementById('submit-status');
  form?.addEventListener('submit', async (e) => {
    e.preventDefault();
    const fd = new FormData(form);
    status!.textContent = '提交中…';
    try {
      const r = await fetch('/api/submit/posts', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          title: fd.get('title'), body: fd.get('body'), author: fd.get('author'),
        }),
      });
      if (!r.ok) throw new Error(String(r.status));
      status!.textContent = '已提交，等待审阅。';
      form.reset();
    } catch {
      status!.textContent = '提交失败，服务暂不可用。';
    }
  });
</script>
```

- [ ] **Step 4: 写 Playwright 测试**

```ts
import { test, expect } from '@playwright/test';

test('投稿页继承主题（暗色下不是白底）', async ({ page }) => {
  await page.addInitScript(() => localStorage.setItem('theme', 'dark'));
  await page.goto('/submit/');
  await expect(page.locator('html')).toHaveAttribute('data-theme', 'dark');
  await expect(page.locator('.mica-card')).toBeVisible();
});

test('后端 500 时显示失败文案', async ({ page }) => {
  await page.route('**/api/submit/**', (r) => r.fulfill({ status: 500 }));
  await page.goto('/submit/');
  await page.fill('input[name="title"]', 't');
  await page.fill('input[name="author"]', 'a');
  await page.fill('textarea[name="body"]', 'b');
  await page.click('button[type="submit"]');
  await expect(page.locator('#submit-status')).toContainText('提交失败');
});
```

- [ ] **Step 5: 运行确认通过**

Run: `pnpm build && node scripts/assert/submit.mjs && pnpm exec playwright test tests/e2e/submit.spec.ts`
Expected: `SUBMIT OK` + 2 passed

- [ ] **Step 6: Commit**

```bash
git add src/pages/submit.astro scripts/assert/submit.mjs tests/e2e/submit.spec.ts
git commit -m "feat(submit): submission page using Base layout and mica card"
```

---

## 阶段五：部署与备案（顺序已修正）

> **v1 致命顺序错误（B 审查）**：v1 把 HTTPS(Task 17) 排在备案(Task 18) **之前**。大陆服务器在备案号下来前，80/443 会被接入商阻断，**certbot 的 HTTP-01 挑战必然失败**。
>
> **v2 正确顺序**：服务器基线 → Nginx(HTTP) → 部署 → **备案** → DNS 解析 → 证书(HTTPS)。

### Task 16: 服务器基线（含内存护栏与模块探测）

**Files:** Create `deploy/scripts/00-baseline.sh`

**约束依据**：2 GiB + Artalk(~80MB) + submitd(~25MB) + Nginx(~30MB) ≈ 135MB，**必须主动防 OOM**。

- [ ] **Step 1: 写脚本**

```bash
#!/usr/bin/env bash
set -euo pipefail

apt-get update
apt-get install -y nginx certbot python3-certbot-nginx \
  libnginx-mod-http-brotli-static libnginx-mod-http-brotli-filter

# 2 GB swap：2 GiB 机器的 OOM 保险丝
if ! swapon --show | grep -q '/swapfile'; then
  fallocate -l 2G /swapfile && chmod 600 /swapfile && mkswap /swapfile && swapon /swapfile
  grep -q '/swapfile' /etc/fstab || echo '/swapfile none swap sw 0 0' >> /etc/fstab
fi
sysctl -w vm.swappiness=10
grep -q 'vm.swappiness' /etc/sysctl.conf || echo 'vm.swappiness=10' >> /etc/sysctl.conf

# 服务内存硬限（只用 MemoryMax，不用 MemoryHigh：软限会触发 Go 运行时频繁 GC）
mkdir -p /etc/systemd/system/artalk.service.d /etc/systemd/system/submitd.service.d
cat > /etc/systemd/system/artalk.service.d/limits.conf <<'EOF'
[Service]
MemoryMax=256M
Restart=on-failure
EOF
cat > /etc/systemd/system/submitd.service.d/limits.conf <<'EOF'
[Service]
MemoryMax=96M
Restart=on-failure
EOF

# 部署目录与数据目录
mkdir -p /var/www/lancycelestia/releases /var/lib/submitd /var/lib/artalk
chown -R www-data:www-data /var/www/lancycelestia
chown -R artalk:artalk /var/lib/artalk 2>/dev/null || true
chown -R www-data:www-data /var/lib/submitd
```

- [ ] **Step 2: 用户执行并回贴三项输出**

```bash
nginx -V 2>&1 | tr ' ' '\n' | grep -E 'brotli|gzip_static|http_v3' || echo 'NONE'
free -h && swapon --show
```

- [ ] **Step 3: 依据输出决定 Nginx 模块指令**

| 输出 | 处理 |
|---|---|
| 含 `brotli` | Task 17 启用 `brotli_static on;` |
| **无** `gzip_static` | Task 16 **不得**写 `gzip_static on;`（否则 `nginx -t` 报 unknown directive） |
| **无** `http_v3` | 预期内（台账 4.13），配置中**不得**出现 `quic`/`http3` |
| `swapon` 为空 | Step 1 失败，**回到 Step 1 重跑**，不得继续 |

---

### Task 17: Nginx 配置（www/裸域发副本，不做 301）

**Files:** Create `deploy/nginx/blog.conf`

**决策依据**：用户要求 `www` 与裸域**各放一份 blog 页面副本，不做 301**。用 `canonical` 解决重复内容。

- [ ] **Step 1: 写配置（模块指令按 Task 16 输出条件启用）**

```nginx
# ---------- 通用压缩设置 ----------
# 若 Task 16 输出无 gzip_static，删除 gzip_static 行
brotli_static on;
gzip_static on;
brotli_types text/html text/css application/javascript application/json image/svg+xml application/xml;
gzip_types text/html text/css application/javascript application/json image/svg+xml application/xml;

limit_req_zone $binary_remote_addr zone=perip:10m rate=10r/s;

# ---------- blog.lancycelestia.com（主站） ----------
server {
    listen 80;
    server_name blog.lancycelestia.com;
    root /var/www/lancycelestia/current;
    index index.html;

    # Brotli/Gzip 协商
    add_header Vary Accept-Encoding;

    # 带 hash 的静态资源：一年 immutable
    location /_assets/ {
        add_header Cache-Control "public, max-age=31536000, immutable" always;
        try_files $uri =404;
    }

    # HTML：短缓存
    location ~* \.html$ {
        add_header Cache-Control "public, max-age=300, must-revalidate" always;
        try_files $uri =404;
    }

    # 评论：Artalk 反代（注意结尾不带 /，保留 /api/v2 原样）
    location /api/v2/ {
        proxy_pass http://127.0.0.1:23366;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # 投稿：自研 submitd
    location /api/submit/ {
        limit_req zone=perip burst=5 nodelay;
        rewrite ^/api/submit/(.*)$ /$1 break;
        proxy_pass http://127.0.0.1:23367;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        client_max_body_size 64k;
    }

    # 搜索索引（Pagefind）
    location /pagefind/ {
        add_header Cache-Control "public, max-age=3600" always;
        try_files $uri =404;
    }

    location / {
        try_files $uri $uri/ $uri.html =404;
    }

    # 防爬
    location ~ /\. { deny all; }
    if ($http_user_agent ~* (semrush|ahrefs|mj12bot|dotbot)) { return 403; }
}

# ---------- www 与裸域：同一份内容副本，不做跳转 ----------
server {
    listen 80;
    server_name www.lancycelestia.com lancycelestia.com;
    root /var/www/lancycelestia/current;   # 与主站同一份产物
    index index.html;
    add_header Vary Accept-Encoding;
    location /_assets/ { add_header Cache-Control "public, max-age=31536000, immutable" always; try_files $uri =404; }
    location / { try_files $uri $uri/ $uri.html =404; }
    # 注意：此处不得出现 return 301 / rewrite ... https://blog. ，用户明确要求不跳转
}
```

- [ ] **Step 2: 用户执行**

```bash
cp deploy/nginx/blog.conf /etc/nginx/sites-available/blog.conf
ln -sf /etc/nginx/sites-available/blog.conf /etc/nginx/sites-enabled/blog.conf
rm -f /etc/nginx/sites-enabled/default
nginx -t
systemctl reload nginx
```

- [ ] **Step 3: 验证（三条硬性）**

```bash
# 1) brotli 生效
curl -sI -H 'Accept-Encoding: br' https://blog.lancycelestia.com/index.html | grep -i content-encoding
# 2) www 与裸域不得 301 到 blog.
curl -sI http://www.lancycelestia.com | grep -iE '^(HTTP|location)'
curl -sI http://lancycelestia.com | grep -iE '^(HTTP|location)'
# 期望：HTTP/1.1 200，且无 location: https://blog.…
```

---

### Task 18: 部署脚本（tar + scp + 原子软链切换）

**Files:** Create `scripts/deploy.sh`

- [ ] **Step 1: 写脚本**

```bash
#!/usr/bin/env bash
set -euo pipefail
HOST="${DEPLOY_HOST:?set DEPLOY_HOST}"
USER_="${DEPLOY_USER:-deploy}"
TS=$(date +%Y%m%d%H%M%S)
REMOTE=/var/www/lancycelestia

pnpm build && pnpm compress
tar -czf /tmp/dist-$TS.tar.gz -C dist .

scp /tmp/dist-$TS.tar.gz "$USER_@$HOST:/tmp/"

ssh "$USER_@$HOST" bash -s <<EOF
set -euo pipefail
mkdir -p $REMOTE/releases/$TS
tar -xzf /tmp/dist-$TS.tar.gz -C $REMOTE/releases/$TS
chown -R www-data:www-data $REMOTE/releases/$TS
ln -sfn $REMOTE/releases/$TS $REMOTE/current
ls -1dt $REMOTE/releases/* | tail -n +6 | xargs -r rm -rf
rm -f /tmp/dist-$TS.tar.gz
EOF
rm -f /tmp/dist-$TS.tar.gz
echo "deployed $TS"
```

- [ ] **Step 2: 执行并验证零停机**

```bash
DEPLOY_HOST=<服务器IP> bash scripts/deploy.sh
# 另开一个终端持续请求，确认切换瞬间无 404
while true; do curl -s -o /dev/null -w "%{http_code}\n" https://blog.lancycelestia.com/; sleep 0.2; done
```

- [ ] **Step 3: 回滚演练（必须有命令）**

```bash
ssh user@host 'ls -1dt /var/www/lancycelestia/releases/*'
ssh user@host 'ln -sfn /var/www/lancycelestia/releases/<上一版时间戳> /var/www/lancycelestia/current'
```

---

### Task 19: 备案提交（用户操作，含 P1–P5 前置）

**已核实的备案资格**（用户截图确认）：
- 实例 `i-bp1a6bjfudd2u7fqwv1pd` 在**可备案实例管理**列表内
- **免费备案数量 4/5**，即**已用 4、剩 1** ← 足够本项目（1 域名 = 1 码）

| # | 前置 | 状态 |
|---|---|---|
| P1 | 域名已实名认证且满 3 天 | 图2 显示「实名认证成功」✓（**待你确认是否已满 3 天**） |
| P2 | 包年包月 + 公网 IP | 图1/截图 ✓ |
| P3 | 备案服务码 | **剩 1 个，够用** ✓ |
| P4 | DNS 解析指向 `120.27.151.231` | **待你添加（Step 2）** |
| P5 | 网站名称 = 澜汐斋记 | ✓ 已定 |

- [ ] **Step 1: 添加解析（三条记录）**

DNS 控制台 → `lancycelestia.com`：
```
类型 A   主机记录 blog    记录值 120.27.151.231
类型 A   主机记录 www     记录值 120.27.151.231
类型 A   主机记录 @       记录值 120.27.151.231
```
> 三条都要加：备案核验会检查域名可达；`www`/`@` 与 `blog` 内容相同（Task 16 已配同一个 `root`）。

- [ ] **Step 2: 提交备案**

备案控制台 → 首次备案 → 主体（个人/浙江）→ 网站信息：
- 网站名称：`澜汐斋记`
- 域名：`lancycelestia.com`
- 备注（如被问）：`个人读书笔记站点`

- [ ] **Step 3: 短信核验 + 人脸核身**（你本人，注意短信时效）

- [ ] **Step 4: 等待管局审核（约 1–20 工作日）**

期间**不得**改实例规格、释放实例、改解析。

- [ ] **Step 5: 备案通过后填入备案号**

```ts
// src/config/site.ts
icp: '浙ICP备XXXXXXXX号',
```

然后 `pnpm build && pnpm compress && bash scripts/deploy.sh`。

- [ ] **Step 6: 验证备案号已上页脚且非占位符**

```bash
curl -s https://blog.lancycelestia.com/ | grep -o '浙ICP备[0-9]\{8\}号'
# 期望输出真实备案号；若输出为空或含 XXXX，说明未替换或未重新部署
```

---

### Task 20: HTTPS（备案通过后才做）

**前提**：Task 18 备案已通过，且 80 端口可被公网访问。

- [ ] **Step 1: 签发证书**

```bash
certbot --nginx -d blog.lancycelestia.com -d www.lancycelestia.com -d lancycelestia.com \
  --redirect --agree-tos -m <你的邮箱> --no-eff-email
```
> **注意**：`--redirect` 只会把**同域名**的 http 跳到 https，**不会**把 `www` 跳到 `blog`（因为它们是不同的 `server_name` 块）。签发后**必须验证**这一条。

- [ ] **Step 2: 验证约束 1（www/裸域不得 301 到 blog）**

```bash
for h in www.lancycelestia.com lancycelestia.com; do
  echo "== $h =="
  curl -sI http://$h | grep -iE '^(HTTP|location)'
  curl -sIL https://$h | grep -iE '^(HTTP|location)'
done
# 通过标准：Location 只能出现 https://<同一个 host>/…，不得出现 blog.lancycelestia.com
```

- [ ] **Step 3: 验证自动续期**

```bash
certbot renew --dry-run
systemctl list-timers | grep certbot
```

- [ ] **Step 4: SSL Labs 评级**

访问 `https://www.ssllabs.com/ssltest/analyze.html?d=blog.lancycelestia.com`，期望 **A 及以上**。

---

### Task 21: 备份与监控

**Files:** Create `deploy/scripts/backup.sh`

- [ ] **Step 1: 写备份脚本（SQLite + Artalk 数据 + Nginx 配置）**

```bash
#!/usr/bin/env bash
set -euo pipefail
DEST=/var/backups/lancycelestia
mkdir -p "$DEST"
TS=$(date +%Y%m%d)
sqlite3 /var/lib/submitd/submitd.db ".backup '$DEST/submitd-$TS.db'"
tar -czf "$DEST/artalk-$TS.tar.gz" -C /var/lib artalk
tar -czf "$DEST/nginx-$TS.tar.gz" -C /etc nginx/sites-available
find "$DEST" -type f -mtime +14 -delete
```

- [ ] **Step 2: 挂 cron 并验证可恢复**

```bash
echo '0 4 * * * root /opt/lancycelestia/backup.sh' > /etc/cron.d/lancycelestia-backup
# 恢复演练：把备份解压到临时目录并用 sqlite3 打开验证
sqlite3 /tmp/restore-test.db ".tables"
```

---

## 验收清单（全部可执行）

| # | 项 | 命令 | 通过标准 |
|---|---|---|---|
| 1 | 构建 | `pnpm build` | 退出码 0 |
| 2 | 禁用字体 | `node scripts/assert/fonts.mjs` | `FONTS CLEAN` |
| 3 | 设计 token 与三档 | `node scripts/assert/tokens.mjs` | `TOKENS OK` |
| 4 | 玻璃/Mica/Double-Bezel | `node scripts/assert/glass.mjs` | `GLASS OK` |
| 5 | 主题无闪烁/reduced-motion | `node scripts/assert/motion.mjs` | `MOTION OK` |
| 6 | 内容/无死链 | `node scripts/assert/content.mjs` | `CONTENT OK` |
| 7 | 双语/hreflang | `node scripts/assert/i18n.mjs` | `I18N OK` |
| 8 | 搜索索引 | `node scripts/assert/search.mjs` | `SEARCH OK` |
| 9 | RSS/Sitemap/资源策略 | `node scripts/assert/assets.mjs` | `ASSETS OK` |
| 10 | 预压覆盖率与预算 | `node scripts/assert/compress.mjs` | `COMPRESS OK`，index.html.br ≤150KB |
| 11 | 投稿页用 Base | `node scripts/assert/submit.mjs` | `SUBMIT OK` |
| 12 | 四种视觉（人眼 + 自动） | `pnpm exec playwright test tests/e2e/visual.spec.ts` | 亮/暗 × 1440/768/390 六张截图 |
| 12b | **参考质感移植** | `node scripts/assert/reference.mjs && pnpm exec playwright test tests/e2e/reveal.spec.ts` | `REFERENCE OK` + 5 passed（颗粒 / 滚动进场 / reduced-motion / 未采用暖橙） |
| 13 | 动效三档 | `pnpm exec playwright test tests/e2e/motion.spec.ts` | 3 passed |
| 14 | 后端挂掉 | `pnpm exec playwright test tests/e2e/comments.spec.ts` | 3 passed |
| 15 | 投稿失败态 | `pnpm exec playwright test tests/e2e/submit.spec.ts` | 2 passed |
| 16 | 无 JS | `pnpm exec playwright test tests/e2e/nojs.spec.ts` | 1 passed |
| 17 | **www/裸域不跳转** | `curl -sI http://www.lancycelestia.com` | 无 `location: …blog.…` |
| 18 | Brotli 生效 | `curl -sI -H 'Accept-Encoding: br' …` | `content-encoding: br` |
| 19 | HTTPS | SSL Labs | A 及以上 |
| 20 | 备案 | `https://beian.miit.gov.cn/` 查询 | 备案号可查且页脚一致 |

## 审查问题对照表（69 → 处置）

| 来源 | 条数 | 处置 |
|---|---|---|
| A 测试与断言 | 30 | 全部重写断言（T1–T5）；A8 恒真→`motion.mjs` 解析 head；A2 `grep -E "\|"` → 删除，改 Node 脚本；E5 import 链 → Task 2/3 断言编译产物 |
| C 视觉与性能 | 28 | P0-1 `@supports` 正向；P0-2 Bento 改 MicaCard；P0-3 去 `filter:blur`；P0-4/5 reduced-motion 落 CSS 层；P0-6 高光/阴影 token 化；P0-7 字体策略写明系统栈；P1-10 `className`；P1-11 补 span-12；P1-13 `z-index:0`；P1-14 `stroke-dashoffset`；P1-15 `:focus-visible`；P1-16 `aria-live`；P1-17 submit 用 Base；P2-19 补内壳；P2-25 白名单压缩 |
| B 架构与合规 | 11 | 1 备案资格 → **用户实测已排除误判**；2 Artalk 无投稿 → **自研 submitd**；3 `/api/v2` 路径 → Task 14/17；4 HTTPS 顺序 → 重排为 18→19；5 www/裸域 → Task 16 发副本；6 webwallgl 2.66MiB → Task 12 明示体积与费用；8 MemoryMax → Task 15 去掉 MemoryHigh |
