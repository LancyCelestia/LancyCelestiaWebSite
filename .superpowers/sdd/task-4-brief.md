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

