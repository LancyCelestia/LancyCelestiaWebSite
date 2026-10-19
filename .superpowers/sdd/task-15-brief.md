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

