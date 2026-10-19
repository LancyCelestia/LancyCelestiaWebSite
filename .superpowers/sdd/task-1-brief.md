### Task 1: 仓库初始化（含 .gitignore 与构建脚本）

**Files:**
- Create: `.gitignore`, `package.json`, `astro.config.mjs`
- Test: `scripts/assert/build.mjs`

**Interfaces:**
- Produces: `pnpm build` / `pnpm compress` / `pnpm test:assert` 可用；`dist/index.html` 存在

- [ ] **Step 1: 写失败测试 `scripts/assert/build.mjs`**

```js
import { existsSync } from 'node:fs';
const p = 'dist/index.html';
if (!existsSync(p)) { console.error(`MISSING ${p}`); process.exit(1); }
console.log('BUILD OK');
```

- [ ] **Step 2: 运行确认失败**

Run: `node scripts/assert/build.mjs`
Expected: `MISSING dist/index.html`，退出码 1

- [ ] **Step 3: 写 `.gitignore`（v1 缺失，导致 `git add -A` 会提交 dist 与 node_modules）**

```gitignore
node_modules/
dist/
.astro/
*.br
*.gz
.env
.env.*
server/submitd/submitd
server/submitd/*.db
server/submitd/*.db-*
test-results/
playwright-report/
```

- [ ] **Step 4: 初始化并写 `package.json`（v1 声称 `pnpm build` 却从未定义）**

```bash
pnpm init
pnpm add astro@^7.4.1
pnpm add -D pagefind playwright @playwright/test
node -e "
const fs=require('fs');const p=JSON.parse(fs.readFileSync('package.json','utf8'));
p.type='module';
p.scripts={
  dev:'astro dev',
  build:'astro build && pagefind --site dist',
  compress:'node scripts/compress.mjs',
  'test:assert':'node scripts/assert/build.mjs && node scripts/assert/fonts.mjs',
  'test:e2e':'playwright test'
};
fs.writeFileSync('package.json',JSON.stringify(p,null,2));
"
```

- [ ] **Step 5: 写 `astro.config.mjs`（显式 assets 命名，供 immutable 缓存）**

```js
import { defineConfig } from 'astro/config';

export default defineConfig({
  site: 'https://blog.lancycelestia.com',
  build: {
    format: 'directory',
    assets: '_assets',
    inlineStylesheets: 'auto',
  },
  i18n: {
    locales: ['zh', 'en'],
    defaultLocale: 'zh',
    routing: { prefixDefaultLocale: false },
  },
});
```

- [ ] **Step 6: 写最小首页并构建**

`src/pages/index.astro`：
```astro
---
---
<!doctype html>
<html lang="zh"><head><meta charset="utf-8"><title>澜汐斋记</title></head>
<body><h1>澜汐斋记</h1></body></html>
```

Run: `pnpm build`
Expected: 退出码 0，`dist/index.html` 存在

- [ ] **Step 7: 运行测试确认通过**

Run: `pnpm test:assert`
Expected: `BUILD OK`（`fonts.mjs` 在 Task 2 建立，本步先只跑 build.mjs：`node scripts/assert/build.mjs`）

- [ ] **Step 8: Commit**

```bash
git init
git add -A
git commit -m "chore: init astro project with gitignore, build and assert scripts"
```

---

