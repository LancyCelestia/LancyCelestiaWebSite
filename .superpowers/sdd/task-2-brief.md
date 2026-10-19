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

