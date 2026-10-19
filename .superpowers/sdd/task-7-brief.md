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

