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

