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

