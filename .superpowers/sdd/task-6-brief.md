### Task 6: 中英双语路由 + hreflang + canonical

**Files:**
- Create: `src/pages/en/index.astro`, `src/pages/en/blog/[...slug].astro`, `src/pages/en/blog/index.astro`, `src/components/LangSwitch.astro`
- Modify: `src/layouts/Base.astro`（Props 增加 `hreflang` 输出）
- Test: `scripts/assert/i18n.mjs`

**Interfaces:**
- Consumes: `t(key, lang)`、`site.url`
- Produces: `/en/*` 镜像；`<base>` 输出三条 `alternate` link

- [ ] **Step 1: 写失败测试 `scripts/assert/i18n.mjs`**

```js
import { readFileSync, existsSync } from 'node:fs';
const fail = (m) => { console.error(m); process.exit(1); };

for (const p of ['dist/en/index.html', 'dist/index.html']) {
  if (!existsSync(p)) fail(`MISSING ${p}`);
}
const en = readFileSync('dist/en/index.html', 'utf8');
const zh = readFileSync('dist/index.html', 'utf8');

// 1) lang 属性正确
if (!/<html[^>]*lang="en"/.test(en)) fail('EN page lang wrong');
if (!/<html[^>]*lang="zh"/.test(zh)) fail('ZH page lang wrong');

// 2) 三条 hreflang 齐全且 href 精确（v1 只查 hreflang 存在）
const expect = (html, page) => {
  const pairs = [['zh','https://blog.lancycelestia.com/'],
                 ['en','https://blog.lancycelestia.com/en/'],
                 ['x-default','https://blog.lancycelestia.com/']];
  for (const [lang, href] of pairs) {
    const re = new RegExp(`<link[^>]+rel="alternate"[^>]*hreflang="${lang}"[^>]*href="${href}"`);
    const re2 = new RegExp(`<link[^>]+href="${href}"[^>]*hreflang="${lang}"`);
    if (!re.test(html) && !re2.test(html)) fail(`${page}: missing hreflang="${lang}" href="${href}"`);
  }
};
expect(en, 'EN');
expect(zh, 'ZH');

// 3) 英文页 UI 不得含中文（扫 i18n 字典，而非单串黑名单）
const i18nSrc = readFileSync('src/i18n/ui.ts', 'utf8');
const enBlock = i18nSrc.match(/en:\s*\{([\s\S]*?)\}/)?.[1] ?? '';
if (/[\u4e00-\u9fff]/.test(enBlock)) fail('en dictionary contains Chinese characters');

// 4) canonical 必须指向 blog 主域
if (!/<link[^>]+rel="canonical"[^>]+href="https:\/\/blog\.lancycelestia\.com\//.test(zh))
  fail('missing canonical to blog.lancycelestia.com');

console.log('I18N OK');
```

- [ ] **Step 2: 运行确认失败**

Run: `node scripts/assert/i18n.mjs`
Expected: FAIL（`MISSING dist/en/index.html`）

- [ ] **Step 3: 在 `Base.astro` 输出 hreflang 与 canonical**

把 `<head>` 内的 `link` 替换为：
```astro
  <link rel="canonical" href="https://blog.lancycelestia.com/" />
  <link rel="alternate" hreflang="zh" href="https://blog.lancycelestia.com/" />
  <link rel="alternate" hreflang="en" href="https://blog.lancycelestia.com/en/" />
  <link rel="alternate" hreflang="x-default" href="https://blog.lancycelestia.com/" />
```

- [ ] **Step 4: 写 `src/pages/en/index.astro`（全部文案走 `t(_, 'en')`）**

结构与 `src/pages/index.astro` 相同，但 `lang="en"`、`t(k,'en')`，文章取 `data.lang === 'en'`，链接前缀 `/en`。

- [ ] **Step 5: 写 `src/pages/en/blog/[...slug].astro` 与 `en/blog/index.astro`**

英文 `getStaticPaths` 只取 `data.lang === 'en'`，路径 `/en/blog/<slug>/`。

- [ ] **Step 6: 写 `LangSwitch.astro`（在 Nav 右侧）**

```astro
---
interface Props { current: 'zh' | 'en' }
const { current } = Astro.props;
const other = current === 'zh' ? 'en' : 'zh';
const href = other === 'zh' ? '/' : '/en/';
---
<a href={href} class="lang" hreflang={other} lang={other}>{other === 'en' ? 'EN' : '中文'}</a>
```

- [ ] **Step 7: 运行确认通过**

Run: `pnpm build && node scripts/assert/i18n.mjs`
Expected: `I18N OK`

- [ ] **Step 8: 故意破坏验证（T2）**

删掉 `Base.astro` 里 `hreflang="x-default"` 那一行：
Run: `pnpm build && node scripts/assert/i18n.mjs`
Expected: `missing hreflang="x-default"`，退出码 1。**改回。**

- [ ] **Step 9: Commit**

```bash
git add src/pages/en src/components/LangSwitch.astro src/layouts/Base.astro scripts/assert/i18n.mjs
git commit -m "feat(i18n): english mirror with canonical and accurate hreflang"
```

---

