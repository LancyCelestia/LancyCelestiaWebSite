### Task 12: 壁纸入口（CSS 默认 + webwallgl 按需）

**Files:**
- Create: `src/components/Wallpaper.astro`
- Modify: `src/layouts/Base.astro`（可选挂载）
- Test: `tests/e2e/wallpaper.spec.ts`

**决策依据**：用户选 **B3/C**（两者都做）。`webwallgl` 未压缩 **2.66 MiB**，按 0.8 元/GB ≈ **0.0021 元/次**。必须在按钮上明示。

- [ ] **Step 1: 写失败测试 `tests/e2e/wallpaper.spec.ts`**

```ts
import { test, expect } from '@playwright/test';

test('默认不加载 webwallgl（零额外流量）', async ({ page }) => {
  const requests: string[] = [];
  page.on('request', (r) => requests.push(r.url()));
  await page.goto('/');
  await page.waitForTimeout(1500);
  expect(requests.some((u) => u.includes('webwallgl'))).toBe(false);
});

test('按钮标明体积与费用', async ({ page }) => {
  await page.goto('/');
  const btn = page.locator('#wallpaper-enable');
  await expect(btn).toContainText('2.7 MB');
  await expect(btn).toContainText('流量');
});

test('low 档下按钮禁用', async ({ page }) => {
  await page.emulateMedia({ reducedMotion: 'reduce' });
  await page.goto('/');
  await expect(page.locator('#wallpaper-enable')).toBeDisabled();
});
```

- [ ] **Step 2: 运行确认失败**

Run: `pnpm exec playwright test tests/e2e/wallpaper.spec.ts`
Expected: FAIL（`#wallpaper-enable` 不存在）

- [ ] **Step 3: 写 `Wallpaper.astro`**

```astro
---
---
<section class="wallpaper-setup">
  <h2>动态壁纸</h2>
  <p>默认使用 CSS 釉瑚渐变（零流量）。启用 Wallpaper Engine 壁纸需下载约 <strong>2.7 MB</strong> 引擎，按流量计费约 <strong>0.002 元/次</strong>。</p>
  <button id="wallpaper-enable" type="button">启用 WE 壁纸（2.7 MB · 产生流量）</button>
  <p id="wallpaper-status" role="status"></p>
</section>

<script>
  const btn = document.getElementById('wallpaper-enable') as HTMLButtonElement;
  const status = document.getElementById('wallpaper-status')!;
  const root = document.documentElement;

  const syncDisabled = () => {
    btn.disabled = root.dataset.motion === 'low';
    btn.title = btn.disabled ? '低动效档位下不启用壁纸' : '';
  };
  syncDisabled();
  new MutationObserver(syncDisabled).observe(root, { attributes: true, attributeFilter: ['data-motion'] });

  btn.addEventListener('click', async () => {
    btn.disabled = true;
    status.textContent = '正在加载引擎…';
    try {
      // ponytail: 2.66MiB 引擎按需加载；若将来体积失控，改为只保留 CSS 渐变
      const mod: any = await import('webwallgl');
      const canvas = document.createElement('canvas');
      canvas.className = 'wallpaper-canvas';
      document.body.prepend(canvas);
      await mod.mount(canvas, { source: '/wallpapers/default' });
      status.textContent = '壁纸已启用。';
    } catch (e) {
      status.textContent = '引擎加载失败，已保持 CSS 渐变背景。';
      btn.disabled = false;
    }
  });
</script>
```

> `webwallgl` 的真实 API 需在 P3 阶段实测（B 审查指出它的 `main` 指向 UMD 构建）。**若 `import()` 的 API 与 README 不符，按 `ponytail:` 注释降级为仅 CSS 渐变，并在此任务内记录实测结果。**

- [ ] **Step 4: 运行确认通过**

Run: `pnpm build && pnpm exec playwright test tests/e2e/wallpaper.spec.ts`
Expected: 3 passed

- [ ] **Step 5: Commit**

```bash
git add src/components/Wallpaper.astro tests/e2e/wallpaper.spec.ts
git commit -m "feat(wallpaper): opt-in webwallgl with explicit size and cost"
```

---

## 阶段四：后端（Artalk + 自研投稿）

