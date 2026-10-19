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

