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

