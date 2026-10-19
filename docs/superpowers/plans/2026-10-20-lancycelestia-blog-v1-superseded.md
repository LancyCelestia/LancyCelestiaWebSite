# 澜汐斋记 实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 在 `blog.lancycelestia.com` 上线一个客户端渲染四种视觉风格、服务器零渲染、中英双语的个人博客，并完成 ICP 备案。

**Architecture:** 构建期用 Astro 7 生成纯静态产物（HTML/CSS/少量 JS），本地 Windows 完成构建与 Brotli 预压，`tar+scp` 上传后原子软链切换，Nginx 只发文件。评论/注册/主页由 `Artalk`（Go + 纯 Go SQLite）经 Nginx 反代 `/api` 提供。所有动效在客户端 GPU 渲染。

**Tech Stack:** Astro 7.4.1 / Node ≥22.22.0 / 原生 CSS / TypeScript / Nginx（brotli_static）/ Artalk / certbot / webwallgl v3.0.0

**Spec:** `docs/superpowers/specs/2026-10-20-lancycelestia-blog-design.md`

## Global Constraints

- 站点域名：**仅** `blog.lancycelestia.com`；`www` 与裸域**不做任何 301**（用户明确要求）
- 备案网站名称：**澜汐斋记**
- 站内展示名：潮声茶馆 / 澜汐 / 霞月 / LancyCelestia
- Node 版本下限：`>=22.22.0`（Astro 7.4 要求）；本机 v26.7.0
- **禁止**字体依赖 `Inter` / `Roboto` / `Arial` / `Helvetica` / `Open Sans`
- **禁止** `font-display: swap`（会加剧 CLS）；用 `optional` + `size-adjust`
- 中文**默认不加载 Web 字体**，走 Windows 构建机系统栈
- `backdrop-filter` **只用于固定/吸顶元素与小型面板**，不用于滚动容器
- 所有动效仅使用 `transform` / `opacity`，不得动画 `top/left/width/height`
- 三档动效键名：`data-motion="low|mid|high"`
- `prefers-reduced-motion: reduce` → 强制 `low`；用户 localStorage 选择优先级最高
- 构建产物必须自带 `.br` 与 `.gz` 预压文件
- **不得**把创意工坊壁纸打包进产物（许可约束）
- 服务器侧所有操作由**用户**执行脚本，Agent 不持有 SSH 私钥

## Review Focus

1. **首帧主题闪烁**：用户在暗色系统 + 亮色站点间切换时，页面不得先白后暗——`<head>` 内联脚本必须在首次绘制前写入 `data-theme`。
2. **不支持 `backdrop-filter` 的浏览器**（旧 Safari / 受限内核）：面板必须回落到不透明底色，正文仍可读，不得出现文字叠在花纹上的不可读状态。
3. **`prefers-reduced-motion` 用户**：渐变、粒子、壁纸必须全部静止，不得有缓慢漂移。
4. **后端不可用时的评论区块**：Artalk 进程挂掉时，文章页必须正常渲染正文，评论区块显示「评论暂不可用」，而不是整页 500 或白屏。
5. **无 JS 环境 / 搜索引擎爬虫**：正文、标题、导航必须已在 HTML 里，不依赖客户端 JS 注入。

---

## 文件结构

```
lancycelestia.com/
├─ astro.config.mjs              # 站点配置、i18n、sitemap
├─ src/
│  ├─ config/site.ts             # 站点元信息 + 功能开关（唯一真源）
│  ├─ styles/
│  │  ├─ tokens.css              # 颜色/字阶/动效变量（改自 astro-haze）
│  │  ├─ glass.css               # 液态玻璃 + 半模糊（改自 astro-haze）
│  │  ├─ mica.css                # 云母质感 + feTurbulence 噪点
│  │  └─ global.css              # 重置 + 基础排版
│  ├─ components/
│  │  ├─ AuroraBackground.astro  # 动态釉瑚渐变（改自 astro-haze）
│  │  ├─ GlassPanel.astro        # Double-Bezel 面板
│  │  ├─ TideLine.astro          # 签名元素：潮线
│  │  ├─ ThemeToggle.astro       # 明暗切换（无闪烁）
│  │  ├─ MotionSwitch.astro      # 三档动效切换
│  │  ├─ Wallpaper.astro         # 动态壁纸入口（webwallgl）
│  │  ├─ Nav.astro  Footer.astro
│  │  └─ SearchModal.astro       # Pagefind
│  ├─ layouts/Base.astro
│  ├─ pages/
│  │  ├─ index.astro  about.astro  links.astro  projects/index.astro
│  │  ├─ blog/index.astro  blog/[...slug].astro  blog/page/[page].astro
│  │  ├─ tags/index.astro  tags/[tag].astro
│  │  ├─ en/…（镜像）
│  │  ├─ rss.xml.ts  404.astro
│  └─ content/  blog/  projects/
├─ scripts/
│  ├─ compress.mjs               # 生成 .br / .gz
│  └─ deploy.sh                  # tar+scp+原子切换
└─ deploy/nginx/blog.conf
```

---

## 阶段一：地基（本地可跑）

### Task 1: 初始化项目并清理禁用项

**Files:**
- Create: `package.json`, `astro.config.mjs`, `src/pages/index.astro`
- Delete: 无

**Interfaces:**
- Produces: `pnpm build` 可产出 `dist/index.html`

- [ ] **Step 1: 初始化**

```bash
cd "C:/Users/LancyCelestia/Documents/MyWorkspace/lancycelestia.com"
pnpm init
pnpm add astro@^7.4.1
```

- [ ] **Step 2: 写 `astro.config.mjs`**

```js
import { defineConfig } from 'astro/config';

export default defineConfig({
  site: 'https://blog.lancycelestia.com',
  build: { format: 'directory' },
  i18n: {
    locales: ['zh', 'en'],
    defaultLocale: 'zh',
    routing: { prefixDefaultLocale: false },
  },
});
```

- [ ] **Step 3: 写最小首页验证构建**

`src/pages/index.astro`：
```astro
---
---
<html lang="zh"><head><title>澜汐斋记</title></head>
<body><h1>澜汐斋记</h1></body></html>
```

- [ ] **Step 4: 构建验证**

Run: `pnpm exec astro build`
Expected: 输出 `dist/index.html`，退出码 0

- [ ] **Step 5: 断言无禁用字体（不得用 `|| echo`，否则命令报错也会显示 CLEAN）**

Run:
```bash
node -e "
const {execSync}=require('child_process');
const out=execSync('grep -rniE \"inter|roboto|helvetica|arial|open sans\" src dist --include=*.css --include=*.astro --include=*.html || true',{encoding:'utf8'});
if(out.trim()){console.error('BANNED FONT FOUND:\n'+out);process.exit(1);}
console.log('FONTS CLEAN');
"
```
Expected: `FONTS CLEAN`（若 `src`/`dist` 不存在，会先因 Task 1 Step 4 已断言构建成功而不会走到这里）

- [ ] **Step 6: Commit**

```bash
git init && git add -A && git commit -m "chore: init astro project with i18n config"
```

---

### Task 2: 设计 token（四色釉瑚 + 双主题）

**Files:**
- Create: `src/styles/tokens.css`
- Test: `scripts/check-tokens.mjs`

**Interfaces:**
- Produces: CSS 变量 `--glaze-abyss` `--glaze-celadon` `--glaze-coral` `--glaze-orchid` `--glaze-ink` `--panel-blur` `--dur-base` `--ease-glide`

- [ ] **Step 1: 写失败测试**

`scripts/check-tokens.mjs`：
```js
import { readFileSync } from 'node:fs';
const css = readFileSync('src/styles/tokens.css', 'utf8');
const must = ['--glaze-abyss','--glaze-celadon','--glaze-coral','--glaze-orchid','--panel-blur','--ease-glide'];
const missing = must.filter(v => !css.includes(v + ':'));
const bannedFont = /(font-family|--font-sans)[^;]*(Inter|Roboto|Arial|Helvetica)/i.test(css);
if (missing.length) { console.error('MISSING:', missing); process.exit(1); }
if (bannedFont) { console.error('BANNED FONT FOUND'); process.exit(1); }
console.log('TOKENS OK');
```

- [ ] **Step 2: 运行确认失败**

Run: `node scripts/check-tokens.mjs`
Expected: FAIL `ENOENT` 或 `MISSING`

- [ ] **Step 3: 写 `tokens.css`**

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

  --aurora-1: #3E9E8F;
  --aurora-2: #6FA8D8;
  --aurora-3: #9B6BD6;
  --aurora-4: #F2729B;

  --panel-blur: 14px;
  --panel-sat: 150%;
  --panel-bg: color-mix(in oklab, var(--glaze-abyss) 6%, var(--glaze-porcelain) 94%);

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
  --aurora-1: #5FD6C4;
  --aurora-2: #4E8FD0;
  --aurora-3: #B58CF2;
  --aurora-4: #FF8FB4;
  --panel-bg: color-mix(in oklab, var(--glaze-abyss) 14%, var(--glaze-porcelain) 86%);
}

:root[data-motion="low"]  { --panel-blur: 0px; --panel-sat: 100%; }
:root[data-motion="mid"]  { --panel-blur: 14px; --panel-sat: 150%; }
:root[data-motion="high"] { --panel-blur: 24px; --panel-sat: 180%; }
```

- [ ] **Step 4: 运行测试确认通过**

Run: `node scripts/check-tokens.mjs`
Expected: `TOKENS OK`

- [ ] **Step 5: Commit**

```bash
git add src/styles/tokens.css scripts/check-tokens.mjs
git commit -m "feat(design): add glaze color tokens with dual theme and motion tiers"
```

---

### Task 3: 液态玻璃 + 半模糊 + Mica

**Files:**
- Create: `src/styles/glass.css`, `src/styles/mica.css`, `src/components/GlassPanel.astro`
- Test: `scripts/check-glass.mjs`

**Interfaces:**
- Consumes: `--panel-blur` `--panel-sat` `--panel-bg` `--glaze-hairline`
- Produces: `.glass` `.glass-panel` 类；`<GlassPanel>` 组件

- [ ] **Step 1: 写失败测试**

`scripts/check-glass.mjs`：
```js
import { readFileSync } from 'node:fs';
const glass = readFileSync('src/styles/glass.css', 'utf8');
const mica  = readFileSync('src/styles/mica.css', 'utf8');

// 前缀必须在前，标准属性在后（否则 CSS 管线折叠后丢失）
const iPre = glass.indexOf('-webkit-backdrop-filter');
const iStd = glass.indexOf('\n  backdrop-filter');
if (iStd === -1 || iPre === -1 || iStd < iPre) { console.error('PREFIX ORDER WRONG'); process.exit(1); }

// 必须有 fallback
if (!glass.includes('@supports not')) { console.error('NO @supports FALLBACK'); process.exit(1); }

// Mica 噪点必须内联，不得是外部图片请求
if (/url\(["']?https?:/.test(mica)) { console.error('MICA USES EXTERNAL URL'); process.exit(1); }

console.log('GLASS OK');
```

- [ ] **Step 2: 运行确认失败**

Run: `node scripts/check-glass.mjs`
Expected: FAIL ENOENT

- [ ] **Step 3: 写 `glass.css`**

```css
.glass,
.glass-panel {
  background: var(--panel-bg);
  -webkit-backdrop-filter: blur(var(--panel-blur)) saturate(var(--panel-sat));
  backdrop-filter: blur(var(--panel-blur)) saturate(var(--panel-sat));
  border: 1px solid var(--glaze-hairline);
  border-radius: var(--radius-lg);
  box-shadow: 0 24px 80px -32px rgba(7,22,40,.45), inset 0 1px 0 rgba(255,255,255,.14);
}

.glass-panel { position: relative; overflow: hidden; }

/* 液态玻璃高光层 */
.glass-panel::before {
  content: '';
  position: absolute; inset: 0;
  background: linear-gradient(140deg, rgba(255,255,255,.5), transparent 40%);
  mix-blend-mode: screen;
  opacity: .35;
  pointer-events: none;
}

/* 液面高光跟随指针 */
.glass-panel::after {
  content: '';
  position: absolute; inset: 0;
  background: radial-gradient(160px 160px at var(--mx, 50%) var(--my, 50%),
              rgba(255,255,255,.22), transparent 70%);
  opacity: 0;
  transition: opacity var(--dur-fast) var(--ease-glide);
  pointer-events: none;
}
.glass-panel:hover::after { opacity: 1; }

/* 无 backdrop-filter 支持时的兜底：不透明底色，保证可读 */
@supports not ((backdrop-filter: blur(1px)) or (-webkit-backdrop-filter: blur(1px))) {
  .glass, .glass-panel {
    background: color-mix(in oklab, var(--glaze-porcelain) 92%, var(--glaze-abyss) 8%);
  }
}

@media (prefers-reduced-transparency: reduce) {
  .glass, .glass-panel { background: var(--glaze-porcelain); backdrop-filter: none; }
}
```

- [ ] **Step 4: 写 `mica.css`（内联噪点，零外部请求）**

```css
/* Mica：噪点用内联 SVG feTurbulence，约 300 字节，零图片请求 */
.mica {
  position: relative;
  background: var(--panel-bg);
}
.mica::before {
  content: '';
  position: absolute; inset: 0;
  pointer-events: none;
  opacity: .035;
  mix-blend-mode: overlay;
  background-image: url("data:image/svg+xml;utf8,<svg xmlns='http://www.w3.org/2000/svg' width='160' height='160'><filter id='n'><feTurbulence type='fractalNoise' baseFrequency='0.8' numOctaves='3'/></filter><rect width='160' height='160' filter='url(%23n)'/></svg>");
}
```

- [ ] **Step 5: 写 `GlassPanel.astro`**

```astro
---
interface Props { as?: string; class?: string; }
const { as = 'div', class: cls = '' } = Astro.props;
const Tag = as as any;
---
<Tag class:list={['glass-panel', 'mica', cls]}><slot /></Tag>

<script>
  // 单一委托监听器：避免每个面板各挂一个 pointermove 处理器
  // （页面十几个面板 × 每次移动 = 十几个回调同帧执行，低端机会掉帧）
  let raf = 0;
  let last: { el: HTMLElement; x: number; y: number } | null = null;

  document.addEventListener('pointermove', (e) => {
    const el = (e.target as HTMLElement | null)?.closest<HTMLElement>('.glass-panel');
    if (!el) return;
    const r = el.getBoundingClientRect();
    last = { el, x: e.clientX - r.left, y: e.clientY - r.top };
    if (raf) return;
    raf = requestAnimationFrame(() => {
      raf = 0;
      if (last) {
        last.el.style.setProperty('--mx', `${last.x}px`);
        last.el.style.setProperty('--my', `${last.y}px`);
      }
    });
  }, { passive: true });
</script>
```

- [ ] **Step 6: 运行测试**

Run: `node scripts/check-glass.mjs`
Expected: `GLASS OK`

- [ ] **Step 7: Commit**

```bash
git add src/styles/glass.css src/styles/mica.css src/components/GlassPanel.astro scripts/check-glass.mjs
git commit -m "feat(ui): liquid glass panel, mica noise, non-blur fallback"
```

---

### Task 4: 动态釉瑚渐变 + 三档动效 + 无闪烁主题

**Files:**
- Create: `src/components/AuroraBackground.astro`, `src/components/MotionSwitch.astro`, `src/components/ThemeToggle.astro`, `src/layouts/Base.astro`
- Test: `scripts/check-motion.mjs`

**Interfaces:**
- Produces: `<Base title>` 布局；`initMotion()` / `initTheme()`；`data-motion` `data-theme`

- [ ] **Step 1: 写失败测试**

`scripts/check-motion.mjs`：
```js
import { readFileSync } from 'node:fs';
const base = readFileSync('src/layouts/Base.astro', 'utf8');
const aur  = readFileSync('src/components/AuroraBackground.astro', 'utf8');

// 主题必须在 <body> 前由内联脚本写入，避免首帧闪烁
const headIdx = base.indexOf('data-theme');
const bodyIdx = base.indexOf('<body');
if (headIdx === -1 || bodyIdx === -1 || headIdx > bodyIdx) { console.error('THEME NOT SET BEFORE BODY'); process.exit(1); }

// reduced-motion 必须强制 low
if (!/prefers-reduced-motion/.test(base)) { console.error('NO REDUCED MOTION SUPPORT'); process.exit(1); }

// 动效不得动画布局属性
const banned = /animation[^;]*\b(top|left|width|height)\b/;
if (banned.test(aur)) { console.error('ANIMATES LAYOUT PROPERTY'); process.exit(1); }

console.log('MOTION OK');
```

- [ ] **Step 2: 运行确认失败**

Run: `node scripts/check-motion.mjs`
Expected: FAIL ENOENT

- [ ] **Step 3: 写 `AuroraBackground.astro`（含 `data-motion` 驱动的三档降级）**

```astro
---
interface Props { intensity?: 'low' | 'mid' | 'high' }
const { intensity = 'mid' } = Astro.props;
---
<div class="aurora" data-intensity={intensity} aria-hidden="true">
  <div class="blob b1"></div>
  <div class="blob b2"></div>
  <div class="blob b3"></div>
  <div class="sheen"></div>
</div>

<style>
  .aurora {
    position: fixed; inset: -20%; z-index: -1;
    pointer-events: none; overflow: hidden;
  }
  .blob {
    position: absolute; width: 70%; height: 70%; border-radius: 50%;
    filter: blur(80px); will-change: transform;
  }
  .b1 { background: radial-gradient(circle, var(--aurora-1), transparent 65%); top: -5%; left: -5%; animation: drift1 28s var(--ease-glide) infinite alternate; }
  .b2 { background: radial-gradient(circle, var(--aurora-3), transparent 65%); bottom: -10%; right: -10%; animation: drift2 37s var(--ease-glide) infinite alternate; }
  .b3 { background: radial-gradient(circle, var(--aurora-4), transparent 65%); top: 30%; left: 45%; animation: drift3 45s var(--ease-glide) infinite alternate; }
  .sheen {
    position: absolute; inset: 0; opacity: .18;
    background: conic-gradient(from 0deg, transparent, var(--aurora-2), transparent 40%);
    animation: spin 60s linear infinite;
  }
  @keyframes drift1 { from { transform: translate3d(0,0,0) scale(1); } to { transform: translate3d(8%,6%,0) scale(1.15); } }
  @keyframes drift2 { from { transform: translate3d(0,0,0) scale(1.1); } to { transform: translate3d(-10%,-7%,0) scale(.95); } }
  @keyframes drift3 { from { transform: translate3d(0,0,0) scale(.9); } to { transform: translate3d(-12%,8%,0) scale(1.2); } }
  @keyframes spin { to { transform: rotate(360deg); } }

  /* 三档：low 全静止；mid 去掉第三层与 conic；high 全开 */
  :root[data-motion="low"] .aurora .blob,
  :root[data-motion="low"] .aurora .sheen,
  :root[data-motion="mid"] .aurora .b3,
  :root[data-motion="mid"] .aurora .sheen { animation: none; }
</style>
```

- [ ] **Step 4: 写 `Base.astro`（内联主题脚本在 `<body>` 之前）**

```astro
---
import '../styles/tokens.css';
import '../styles/glass.css';
import '../styles/mica.css';
import AuroraBackground from '../components/AuroraBackground.astro';
interface Props { title: string }
const { title } = Astro.props;
---
<!doctype html>
<html lang="zh" data-theme="light" data-motion="mid">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>{title}</title>
  <script is:inline>
    (function () {
      var r = document.documentElement;
      try { var t = localStorage.getItem('theme'); if (t) r.dataset.theme = t; } catch (e) {}
      try { var m = localStorage.getItem('motion'); if (m) r.dataset.motion = m; } catch (e) {}
      var reduce = matchMedia('(prefers-reduced-motion: reduce)').matches;
      var weak = (navigator.hardwareConcurrency || 8) <= 4 || (navigator.deviceMemory || 8) <= 4;
      var noBlur = !(CSS.supports('backdrop-filter','blur(1px)') || CSS.supports('-webkit-backdrop-filter','blur(1px)'));
      if (!localStorage.getItem('motion')) {
        if (reduce || noBlur) r.dataset.motion = 'low';
        else if (weak) r.dataset.motion = 'mid';
        else r.dataset.motion = 'high';
      } else if (reduce) {
        r.dataset.motion = 'low';
      }
    })();
  </script>
</head>
<body>
  <AuroraBackground intensity="high" />
  <slot />
</body>
</html>
```

- [ ] **Step 5: 写 `MotionSwitch.astro` 与 `ThemeToggle.astro`**

```astro
---
---
<div class="switches">
  <button id="motion-btn" aria-label="切换动效档位">动效：中</button>
  <button id="theme-btn" aria-label="切换明暗">明暗</button>
</div>
<script>
  const ORDER = ['low', 'mid', 'high'] as const;
  const LABEL = { low: '低', mid: '中', high: '高' };
  const mBtn = document.getElementById('motion-btn')!;
  const tBtn = document.getElementById('theme-btn')!;
  const root = document.documentElement;

  const sync = () => {
    mBtn.textContent = `动效：${LABEL[root.dataset.motion as keyof typeof LABEL] ?? '中'}`;
  };
  mBtn.addEventListener('click', () => {
    const i = ORDER.indexOf(root.dataset.motion as any);
    const next = ORDER[(i + 1) % ORDER.length];
    root.dataset.motion = next;
    localStorage.setItem('motion', next);
    sync();
  });
  tBtn.addEventListener('click', () => {
    const next = root.dataset.theme === 'dark' ? 'light' : 'dark';
    root.dataset.theme = next;
    localStorage.setItem('theme', next);
  });
  sync();
</script>
```

- [ ] **Step 6: 运行测试**

Run: `node scripts/check-motion.mjs`
Expected: `MOTION OK`

- [ ] **Step 7: Commit**

```bash
git add src/components src/layouts src/styles
git commit -m "feat(ui): aurora gradient, motion tiers, no-flash theme"
```

---

## 阶段二：内容与页面

### Task 5: 内容集合 + 主页 Bento 布局

**Files:** Create `src/content.config.ts`, `src/content/blog/hello.md`, `src/content/blog/hello.en.md`, `src/pages/index.astro`

**Interfaces:** Produces `getCollection('blog')` 返回 `{ data: { title, lang, date, tags } }`

- [ ] **Step 1: 写失败测试** `scripts/check-content.mjs`：构建后断言 `dist/index.html` 含文章标题、含 `<html lang="zh">`
- [ ] **Step 2: 运行确认失败**
- [ ] **Step 3: 写 `content.config.ts`**

```ts
import { defineCollection, z } from 'astro:content';
import { glob } from 'astro/loaders';

const blog = defineCollection({
  loader: glob({ pattern: '**/*.md', base: './src/content/blog' }),
  schema: z.object({
    title: z.string(),
    lang: z.enum(['zh', 'en']).default('zh'),
    date: z.coerce.date(),
    tags: z.array(z.string()).default([]),
    draft: z.boolean().default(false),
  }),
});
export const collections = { blog };
```

- [ ] **Step 4: 写示例文章 `src/content/blog/hello.md`**

```markdown
---
title: 开斋第一篇
lang: zh
date: 2026-10-20
tags: [随记]
---

潮声起于岸，文字落于纸。这里记下一些想法。
```

英文版 `src/content/blog/hello.en.md` 同结构，`title: The first entry`、`lang: en`。

- [ ] **Step 5: 写 Bento 主页 `src/pages/index.astro`**

```astro
---
import Base from '../layouts/Base.astro';
import GlassPanel from '../components/GlassPanel.astro';
import { getCollection } from 'astro:content';

const posts = (await getCollection('blog', ({ data }) => data.lang === 'zh' && !data.draft))
  .sort((a, b) => b.data.date.valueOf() - a.data.date.valueOf())
  .slice(0, 5);
---
<Base title="澜汐斋记 · 潮声茶馆">
  <main class="bento">
    <section class="hero">
      <p class="eyebrow">潮声茶馆</p>
      <h1>澜汐</h1>
      <p class="sub">霞月 · LancyCelestia</p>
    </section>

    <GlassPanel class="cell span-6">
      <h2>最新文章</h2>
      <ul>
        {posts.map((p) => (
          <li><a href={`/blog/${p.id}/`}>{p.data.title}</a></li>
        ))}
      </ul>
    </GlassPanel>

    <GlassPanel class="cell span-3">
      <h2>项目</h2>
      <p><a href="/projects/">chatbot · QQ 聊天机器人</a></p>
    </GlassPanel>

    <GlassPanel class="cell span-3">
      <h2>关于我</h2>
      <p><a href="/about/">澜汐</a></p>
    </GlassPanel>
  </main>
</Base>

<style>
  .bento {
    display: grid;
    grid-template-columns: repeat(12, 1fr);
    gap: 1.5rem;
    max-width: 1200px;
    margin-inline: auto;
    padding: clamp(1rem, 4vw, 4rem);
  }
  .hero { grid-column: span 12; min-height: 40dvh; display: grid; align-content: center; }
  .eyebrow { font-size: .75rem; letter-spacing: .2em; text-transform: uppercase; color: var(--glaze-ink-soft); }
  h1 { font-size: var(--text-h1); line-height: 1.05; margin: .5rem 0; }
  .span-6 { grid-column: span 6; }
  .span-3 { grid-column: span 3; }
  .cell { padding: 1.75rem; }

  @media (max-width: 768px) {
    .bento { grid-template-columns: 1fr; padding-inline: 1rem; }
    .hero, .span-6, .span-3 { grid-column: span 1; }
  }
</style>
```

- [ ] **Step 6: 构建 + 跑 `check-content.mjs` 确认通过**

Run: `pnpm exec astro build && node scripts/check-content.mjs`
Expected: `CONTENT OK`
- [ ] **Step 6: Commit**

---

### Task 6: 中英双语路由

**Files:** Create `src/i18n/ui.ts`, `src/pages/en/index.astro`, `src/pages/en/blog/index.astro`, `src/components/LangSwitch.astro`

**Interfaces:** Consumes `getCollection('blog')` 的 `lang` 字段；Produces `t(key, lang)` 与 `<LangSwitch />`

- [ ] **Step 1: 写失败测试** `scripts/check-i18n.mjs`

```js
import { readFileSync, existsSync } from 'node:fs';
if (!existsSync('dist/en/index.html')) { console.error('MISSING dist/en/index.html'); process.exit(1); }
const en = readFileSync('dist/en/index.html', 'utf8');
const zh = readFileSync('dist/index.html', 'utf8');
if (!/<html[^>]*lang="en"/.test(en)) { console.error('EN PAGE LANG WRONG'); process.exit(1); }
if (!/<html[^>]*lang="zh"/.test(zh)) { console.error('ZH PAGE LANG WRONG'); process.exit(1); }
// 英文页不得混入中文 UI 串（内容文章除外）
if (/最新文章/.test(en)) { console.error('UNTRANSLATED UI STRING IN EN PAGE'); process.exit(1); }
// 必须声明 hreflang，供搜索引擎识别双语
if (!/hreflang="en"/.test(zh) || !/hreflang="zh"/.test(en)) { console.error('MISSING HREFLANG'); process.exit(1); }
console.log('I18N OK');
```

- [ ] **Step 2: 运行确认失败**

Run: `node scripts/check-i18n.mjs`
Expected: FAIL `MISSING dist/en/index.html`

- [ ] **Step 3: 写 `src/i18n/ui.ts`（唯一文案真源）**

```ts
export const ui = {
  zh: { latest: '最新文章', projects: '项目', about: '关于我', tags: '标签', search: '搜索', submit: '投稿', readMore: '阅读全文' },
  en: { latest: 'Latest posts', projects: 'Projects', about: 'About', tags: 'Tags', search: 'Search', submit: 'Submit', readMore: 'Read more' },
} as const;

export type Lang = keyof typeof ui;
export const t = (key: keyof typeof ui.zh, lang: Lang) => ui[lang][key];
```

- [ ] **Step 4: 实现 en 镜像页，并在 `<Base>` 传入 `lang` 与 `hreflang`**

`src/pages/en/index.astro` 用 `t('latest','en')` 等取文案，**不得硬编码中文**；在 `<Base>` 的 `<head>` 输出：

```astro
<link rel="alternate" hreflang="zh" href="https://blog.lancycelestia.com/" />
<link rel="alternate" hreflang="en" href="https://blog.lancycelestia.com/en/" />
<link rel="alternate" hreflang="x-default" href="https://blog.lancycelestia.com/" />
```

- [ ] **Step 5: 测试通过**

Run: `pnpm exec astro build && node scripts/check-i18n.mjs`
Expected: `I18N OK`

- [ ] **Step 6: Commit**

```bash
git add src/i18n src/pages/en src/components/LangSwitch.astro scripts/check-i18n.mjs
git commit -m "feat(i18n): add en mirror with hreflang and single-source copy"
```

---

### Task 7: Pagefind 站内搜索

**Files:** Create `src/components/SearchModal.astro`；Modify `package.json` 加 `pagefind` 依赖

- [ ] **Step 1: 写失败测试**：构建后断言 `dist/pagefind/` 目录存在
- [ ] **Step 2: 运行确认失败**
- [ ] **Step 3: `pnpm add -D pagefind`，postbuild 执行 `pagefind --site dist`，实现 ⌘K 弹层**
- [ ] **Step 4: 测试通过**
- [ ] **Step 5: Commit**

---

### Task 8: RSS / Sitemap / 404 / 备案页脚

**Files:** Create `src/pages/rss.xml.ts`, `src/components/Footer.astro`；Modify `astro.config.mjs` 加 sitemap

- [ ] **Step 1: 写失败测试**：断言 `dist/rss.xml` 与 `dist/404.html` 存在；Footer 含「澜汐斋记」与 `beian.miit.gov.cn` 链接占位
- [ ] **Step 2: 运行确认失败**
- [ ] **Step 3: 实现**
- [ ] **Step 4: 测试通过**
- [ ] **Step 5: Commit**

---

## 阶段三：视觉增强

### Task 9: 潮线签名元素

**Files:** Create `src/components/TideLine.astro`

- [ ] **Step 1: 写失败测试**：断言组件渲染出 `<svg>` 且含 `stroke-dasharray` 动画类，且 `data-motion="low"` 时无 animation
- [ ] **Step 2: 运行确认失败**
- [ ] **Step 3: 实现 SVG 细线 + CSS 动画（仅 transform/opacity）**
- [ ] **Step 4: 测试通过**
- [ ] **Step 5: Commit**

---

### Task 10: 壁纸入口（webwallgl）

**Files:** Create `src/components/Wallpaper.astro`

**Interfaces:** Produces `<Wallpaper>`；默认**不加载**任何壁纸资源

- [ ] **Step 1: 写失败测试**：断言构建产物**不含**任何 `.pkg` 文件与第三方壁纸资源（许可约束）；断言组件在 `data-motion="low"` 时不初始化
- [ ] **Step 2: 运行确认失败**
- [ ] **Step 3: 懒加载 `webwallgl`：仅在用户主动点击「启用动态壁纸」后 `import('webwallgl')` 并 `mount()`；未启用时零加载**
- [ ] **Step 4: 测试通过**
- [ ] **Step 5: Commit**

---

## 阶段四：构建优化

### Task 11: Brotli/Gzip 预压 + 体积基线

**Files:** Create `scripts/compress.mjs`

- [ ] **Step 1: 写失败测试** `scripts/check-compress.mjs`：断言 `dist/index.html.br` 与 `.gz` 同时存在，且 `.br` 体积 < 原文件
- [ ] **Step 2: 运行确认失败**
- [ ] **Step 3: 用 Node `zlib.brotliCompressSync`（quality 11）与 `gzipSync`（level 9）遍历 dist 生成预压文件**
- [ ] **Step 4: 测试通过**
- [ ] **Step 5: 输出体积报告到 `docs/build-size.md`（实测值，不用业界传说）**
- [ ] **Step 6: Commit**

---

## 阶段五：后端（Artalk）

### Task 12: 部署 Artalk 并反代

**Files:** Create `deploy/systemd/artalk.service`, `deploy/nginx/blog.conf`

- [ ] **Step 1: 写失败测试**：本地 `curl -I http://127.0.0.1:23366/` 前应超时；部署后返回 200
- [ ] **Step 2: 在服务器（用户执行）安装 Artalk 二进制、建 SQLite 库、写 systemd 单元**
- [ ] **Step 3: Nginx `location /api/ { proxy_pass http://127.0.0.1:23366/; }`**
- [ ] **Step 4: 验证 `/api/` 返回 200，且 **Artalk 停掉时文章页仍正常渲染**，评论区显示「评论暂不可用」**
- [ ] **Step 5: Commit**

---

### Task 13: 投稿能力 + 评论降级（补 Artalk 缺口）

**Files:** Create `src/pages/submit.astro`, `src/components/Comments.astro`

- [ ] **Step 1: 写失败测试** `scripts/check-comments.mjs`

```js
import { readFileSync } from 'node:fs';
const c = readFileSync('src/components/Comments.astro', 'utf8');
// 后端不可用时不得让整页失败：必须有 try/catch 与降级文案
if (!/catch/.test(c)) { console.error('NO CATCH'); process.exit(1); }
if (!/评论暂不可用/.test(c)) { console.error('NO FALLBACK COPY'); process.exit(1); }
// 不得在无后端时阻塞渲染
if (/await fetch/.test(c)) { console.error('BLOCKING FETCH AT RENDER TIME'); process.exit(1); }
console.log('COMMENTS OK');
```

- [ ] **Step 2: 运行确认失败**

Run: `node scripts/check-comments.mjs`
Expected: FAIL ENOENT

- [ ] **Step 3: 写 `Comments.astro`（客户端探测，失败则降级）**

```astro
---
---
<section class="comments" data-endpoint="/api/">
  <h2>评论</h2>
  <div id="comments-slot">
    <p class="muted">正在加载评论…</p>
  </div>
</section>

<script>
  const slot = document.getElementById('comments-slot')!;
  const ctrl = new AbortController();
  const timer = setTimeout(() => ctrl.abort(), 4000);

  fetch('/api/', { method: 'GET', signal: ctrl.signal })
    .then((r) => {
      if (!r.ok) throw new Error(`HTTP ${r.status}`);
      return import('artalk');
    })
    .then(() => { slot.innerHTML = '<div id="artalk-root"></div>'; })
    .catch(() => {
      slot.innerHTML = '<p class="muted">评论暂不可用，稍后刷新试试。</p>';
    })
    .finally(() => clearTimeout(timer));
</script>
```

- [ ] **Step 4: 运行测试通过**

Run: `node scripts/check-comments.mjs`
Expected: `COMMENTS OK`

- [ ] **Step 5: 手写 `src/pages/submit.astro`（未登录不显示表单）**

```astro
---
---
<html lang="zh">
  <head><title>投稿 · 澜汐斋记</title></head>
  <body>
    <main class="glass-panel" style="padding:2rem;max-width:640px;margin:4rem auto">
      <h1>投稿</h1>
      <div id="gate"><p>请先登录后再投稿。</p><a href="/auth/login">去登录</a></div>
      <form id="submit-form" hidden>
        <label>标题 <input name="title" required maxlength="120" /></label>
        <label>正文 <textarea name="body" required rows="12"></textarea></label>
        <button type="submit">提交审阅</button>
      </form>
    </main>
    <script>
      fetch('/api/users/me', { credentials: 'include' })
        .then((r) => (r.ok ? r.json() : null))
        .then((u) => {
          if (!u) return;
          document.getElementById('gate')!.hidden = true;
          document.getElementById('submit-form')!.hidden = false;
        })
        .catch(() => {});
    </script>
  </body>
</html>
```

- [ ] **Step 6: 测试通过 + Commit**

```bash
git add src/components/Comments.astro src/pages/submit.astro scripts/check-comments.mjs
git commit -m "feat: comments with graceful degradation and gated submission page"
```

---

## 阶段六：部署与备案

### Task 14: 服务器基线脚本（含内存护栏）

**Files:** Create `deploy/scripts/00-baseline.sh`

**约束依据**：2 GiB 内存 + Artalk 常驻 50–80 MB + Nginx ≈ 100 MB，**必须主动防 OOM**，不能靠运气。

- [ ] **Step 1: 写脚本**

```bash
#!/usr/bin/env bash
set -euo pipefail

apt-get update
apt-get install -y nginx rsync certbot python3-certbot-nginx \
  libnginx-mod-http-brotli-static libnginx-mod-http-brotli-filter

# 1) 2 GB swap —— 2 GiB 机器的 OOM 保险丝
if ! swapon --show | grep -q '/swapfile'; then
  fallocate -l 2G /swapfile
  chmod 600 /swapfile
  mkswap /swapfile
  swapon /swapfile
  grep -q '/swapfile' /etc/fstab || echo '/swapfile none swap sw 0 0' >> /etc/fstab
fi

# 2) 降低 swap 使用倾向（有 swap 但别积极换页，Io 更省）
sysctl -w vm.swappiness=10
grep -q 'vm.swappiness' /etc/sysctl.conf || echo 'vm.swappiness=10' >> /etc/sysctl.conf

# 3) Artalk 内存上限（systemd 层面硬限制，防止它吃爆机器）
mkdir -p /etc/systemd/system/artalk.service.d
cat > /etc/systemd/system/artalk.service.d/limits.conf <<'EOF'
[Service]
MemoryMax=256M
MemoryHigh=192M
Restart=on-failure
EOF

# 4) 部署目录
mkdir -p /var/www/lancycelestia/releases
chown -R www-data:www-data /var/www/lancycelestia
```

- [ ] **Step 2: **用户执行**，回贴三项输出**

```bash
nginx -V 2>&1 | tr ' ' '\n' | grep -E 'brotli|gzip_static|http_v3' || echo 'NONE'
free -h && swapon --show
```

- [ ] **Step 3: 依据输出判定**

| 输出情况 | 处理 |
|---|---|
| 有 `brotli` | 启用 `brotli_static on;` |
| 无 `gzip_static` | Nginx 里**不写** `gzip_static on;`，否则 `nginx -t` 直接报 unknown directive |
| 无 `http_v3` | 预期之内（台账 4.13），配置里**不得出现** `quic` / `http3` 字样 |
| `swapon` 为空 | 说明 Step 1 失败，**回到 Step 1 重跑**，不得继续 |

### Task 15: Nginx 配置

**Files:** Create `deploy/nginx/blog.conf`

- [ ] **Step 1: 配置** `server_name blog.lancycelestia.com;`、`root /var/www/lancycelestia/current;`、`brotli_static on;`、`gzip_static on;`、`location /assets/ { expires 1y; add_header Cache-Control "public, immutable"; }`、`location /api/ { proxy_pass … }`、`limit_req`
- [ ] **Step 2: `nginx -t` 通过**
- [ ] **Step 3: `curl -H 'Accept-Encoding: br' -I https://…/index.html` 断言 `content-encoding: br`**

### Task 16: 部署脚本（tar + scp + 原子切换）

**Files:** Create `scripts/deploy.sh`

- [ ] **Step 1: 本地 `tar -czf` → `scp` → 服务器解包到 `releases/<ts>` → `ln -sfn` 切换 → 保留最近 5 个**
- [ ] **Step 2: 演练一次，断言切换期间 `curl` 无 404**
- [ ] **Step 3: 回滚演练：`ln -sfn` 指回上一版**

### Task 17: HTTPS（certbot 自动续期）

- [ ] **Step 1: `certbot --nginx -d blog.lancycelestia.com`**（用户执行）
- [ ] **Step 2: `certbot renew --dry-run` 通过**
- [ ] **Step 3: 断言 HTTP 301 到 HTTPS；SSL Labs A 级**

### Task 18: 备案提交（用户操作）

**前置条件（缺一不可，按顺序完成后再提交）**

| # | 前置 | 为什么 |
|---|---|---|
| P1 | 域名已完成**实名认证**且满 **3 天**以上 | 未实名/未满期，备案系统会直接拦截 |
| P2 | 服务器为**包年包月**（图1 显示「包年包月」✓） | 按量付费实例不发放备案服务码 |
| P3 | 已申请到**备案服务码** | 无服务码无法提交 |
| P4 | DNS 解析指向本 ECS 的公网 IP（`120.27.151.231`） | 管局会核查解析归属 |
| P5 | 网站名称：**澜汐斋记**（4 汉字、无禁用词） | 见台账 1.1–1.3 |

- [ ] **Step 1: 核对 P1–P3**

在阿里云备案控制台 →「备案服务码」查看额度。**（待补充：你的额度现状）**若无额度，按控制台提示申请。

- [ ] **Step 2: 添加解析**

DNS 控制台 → `lancycelestia.com` → 添加记录：
```
类型 A    主机记录 blog    记录值 120.27.151.231    TTL 600
```
（`www` 与 `@` **不添加**——按你的要求不做跳转）

- [ ] **Step 3: 提交备案**

备案控制台 → 首次备案 → 主体（个人 / 浙江）→ 网站信息：
- 网站名称：`澜汐斋记`
- 域名：`lancycelestia.com`（子域 `blog.` 属同一主域，一般无需单列，提交时向客服确认一次）
- 备注（如被要求说明）：`个人读书笔记站点`

- [ ] **Step 4: 短信核验 + 人脸核身**

由你本人完成；核验短信有**时效**，需当场处理。

- [ ] **Step 5: 等待管局审核（约 1–20 工作日）**

期间**不得**变更实例规格、释放实例、修改域名解析。

- [ ] **Step 6: 通过后在页脚悬挂备案号**

`src/components/Footer.astro`：
```astro
<a href="https://beian.miit.gov.cn/" target="_blank" rel="noopener noreferrer">
  浙ICP备XXXXXXXX号
</a>
```
重新 `pnpm build && pnpm compress` 并部署。

---

## 验收清单

| 项 | 命令/方法 | 通过标准 |
|---|---|---|
| 构建 | `pnpm build` | 退出码 0 |
| 禁用字体 | `grep -rniE "inter\|roboto\|helvetica\|arial" dist` | 无输出 |
| 四种视觉 | 桌面/平板/手机截图 | 釉瑚渐变、液态玻璃、半模糊、Mica 均可见 |
| 三档动效 | 切换 + 系统降级 | 差异明显；reduced-motion 全静止 |
| 无 JS | 禁用 JS 打开 | 正文/导航仍在 |
| 后端挂掉 | `systemctl stop artalk` | 文章页正常，评论区提示 |
| 压缩 | `curl -H 'Accept-Encoding: br' -I` | `content-encoding: br` |
| 体积 | `docs/build-size.md` | 首屏 HTML+CSS+JS ≤ 150 KB (br) |
| HTTPS | SSL Labs | A 及以上 |
| 备案 | 工信部查询 | 备案号可查且页脚一致 |
