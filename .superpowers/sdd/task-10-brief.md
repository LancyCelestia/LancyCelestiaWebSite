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

