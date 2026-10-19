### Task 5: 内容集合 + 主页 Bento + 全套页面

**Files:**
- Create: `src/content.config.ts`, `src/content/blog/{hello.md,hello.en.md}`, `src/config/site.ts`, `src/i18n/ui.ts`
- Create: `src/pages/{index,about,links,404}.astro`, `src/pages/projects/index.astro`, `src/pages/blog/index.astro`, `src/pages/blog/[...slug].astro`, `src/pages/blog/page/[page].astro`, `src/pages/tags/index.astro`, `src/pages/tags/[tag].astro`
- Create: `src/components/{Nav,Footer}.astro`, `src/layouts/Base.astro`（Modify：接入 Nav/Footer）
- Test: `scripts/assert/content.mjs`, `tests/e2e/nojs.spec.ts`

**Interfaces:**
- Consumes: `<Base title lang>`、`<MicaCard className>`、`t(key, lang)`
- Produces: `getCollection('blog')` → `{ id, data: { title, lang, slug, date, tags, draft } }`

> **v1 教训**：`site.ts` 与 `about/links/projects` 页面在 v1 中只有声明、无任务创建 → 首页链到 404。本任务全部补齐。
> **v1 教训**：`p.id` 对 `hello.en.md` 会得到 `hello.en`，URL 带点且与 `/en/` 冲突。**v2 用 schema 的显式 `slug` 字段。**

- [ ] **Step 1: 写失败测试 `scripts/assert/content.mjs`**

```js
import { readFileSync, existsSync } from 'node:fs';
const fail = (m) => { console.error(m); process.exit(1); };

const idx = 'dist/index.html';
if (!existsSync(idx)) fail('MISSING dist/index.html');
const html = readFileSync(idx, 'utf8');

// 1) 正文必须出现在 HTML 里（Review Focus 5）
if (!/<h1[^>]*>[^<]*澜汐/.test(html)) fail('h1 missing or does not contain 澜汐');
if (!/<nav[\s>]/.test(html)) fail('NO <nav> in HTML — nav must not be JS-injected');
if (!/<footer[\s>]/.test(html)) fail('NO <footer> in HTML');

// 2) 草稿不得出现在主页
if (/DRAFT_SENTINEL/.test(html)) fail('draft post leaked to homepage');

// 3) 排序：新文章在前
const a = html.indexOf('开斋第一篇');
const b = html.indexOf('第二篇测试');
if (a === -1) fail('published post 开斋第一篇 not on homepage');
if (b !== -1 && b < a) fail('posts not sorted by date desc');

// 4) 首页所有内部链接的目标页必须真的存在（v1 首页链到 404）
const links = [...html.matchAll(/href="(\/[^"#?]*)"/g)].map((m) => m[1]);
const missing = [];
for (const l of new Set(links)) {
  if (l.startsWith('/_assets')) continue;
  const p = l.endsWith('/') ? `dist${l}index.html` : `dist${l}`;
  const alt = `dist${l.replace(/\/$/, '')}/index.html`;
  if (!existsSync(p) && !existsSync(alt)) missing.push(l);
}
if (missing.length) fail('BROKEN LINKS on homepage:\n' + missing.join('\n'));

console.log('CONTENT OK');
```

- [ ] **Step 2: 运行确认失败**

Run: `node scripts/assert/content.mjs`
Expected: FAIL（`h1 missing` 或 `BROKEN LINKS`）

- [ ] **Step 3: 写 `src/config/site.ts`（v1 声明却从未创建）**

```ts
export const site = {
  name: '澜汐斋记',
  displayName: '潮声茶馆',
  author: '澜汐',
  alias: '霞月',
  aliasEn: 'LancyCelestia',
  url: 'https://blog.lancycelestia.com',
  description: '潮声茶馆 · 澜汐的个人博客',
  icp: '', // 备案通过后填入，如「浙ICP备12345678号」
  features: {
    comments: 'artalk' as 'artalk' | 'off',
    submission: true,
    search: true,
  },
} as const;

export type Lang = 'zh' | 'en';
```

- [ ] **Step 4: 写 `src/i18n/ui.ts`（唯一文案源）**

```ts
export const ui = {
  zh: { latest: '最新文章', projects: '项目', about: '关于我', links: '友链',
        tags: '标签', search: '搜索', submit: '投稿', readMore: '阅读全文',
        archive: '归档', home: '首页', blog: '文章', noPosts: '还没有文章' },
  en: { latest: 'Latest posts', projects: 'Projects', about: 'About',
        links: 'Links', tags: 'Tags', search: 'Search', submit: 'Submit',
        readMore: 'Read more', archive: 'Archive', home: 'Home', blog: 'Blog',
        noPosts: 'No posts yet' },
} as const;

export type Lang = keyof typeof ui;
export const t = (k: keyof typeof ui.zh, lang: Lang): string => ui[lang][k];
```

- [ ] **Step 5: 写 `content.config.ts`（含显式 slug）**

```ts
import { defineCollection, z } from 'astro:content';
import { glob } from 'astro/loaders';

const blog = defineCollection({
  loader: glob({ pattern: '**/*.md', base: './src/content/blog' }),
  schema: z.object({
    title: z.string(),
    lang: z.enum(['zh', 'en']).default('zh'),
    slug: z.string(),
    date: z.coerce.date(),
    tags: z.array(z.string()).default([]),
    draft: z.boolean().default(false),
  }),
});
export const collections = { blog };
```

- [ ] **Step 6: 写示例文章（含草稿）**

`src/content/blog/hello.md`：
```markdown
---
title: 开斋第一篇
lang: zh
slug: hello
date: 2026-10-20
tags: [随记]
---
潮声起于岸，文字落于纸。
```

`src/content/blog/second.md`：
```markdown
---
title: 第二篇测试
lang: zh
slug: second
date: 2026-10-21
tags: [测试]
---
用于验证排序。
```

`src/content/blog/draft.md`：
```markdown
---
title: DRAFT_SENTINEL
lang: zh
slug: draft
date: 2026-10-22
draft: true
---
不应出现在任何列表里。
```

`src/content/blog/hello.en.md`：
```markdown
---
title: The first entry
lang: en
slug: hello
date: 2026-10-20
tags: [notes]
---
Tide rises on the shore.
```

- [ ] **Step 7: 写 `Nav.astro` 与 `Footer.astro`**

`src/components/Nav.astro`：浮动胶囊，使用 `.glass`（**fixed，符合 backdrop-filter 约束**）
```astro
---
import { t } from '../i18n/ui';
import type { Lang } from '../i18n/ui';
import MotionSwitch from './MotionSwitch.astro';
import ThemeToggle from './ThemeToggle.astro';
interface Props { lang?: Lang }
const { lang = 'zh' } = Astro.props;
const base = lang === 'zh' ? '' : '/en';
---
<nav class="glass nav">
  <a href={`${base}/`} class="nav__brand">潮声茶馆</a>
  <a href={`${base}/blog/`}>{t('blog', lang)}</a>
  <a href={`${base}/tags/`}>{t('tags', lang)}</a>
  <a href={`${base}/projects/`}>{t('projects', lang)}</a>
  <a href={`${base}/about/`}>{t('about', lang)}</a>
  <a href={`${base}/links/`}>{t('links', lang)}</a>
  <MotionSwitch />
  <ThemeToggle />
</nav>
<style>
  .nav {
    position: fixed; top: 1.5rem; left: 50%; transform: translateX(-50%);
    z-index: 100; display: flex; gap: 1rem; align-items: center;
    padding: .625rem 1.25rem; border-radius: 9999px; width: max-content;
    max-width: calc(100vw - 2rem); flex-wrap: wrap;
  }
  .nav a { color: var(--glaze-ink); text-decoration: none; font-size: .9rem; }
  .nav__brand { font-family: var(--font-display); font-size: 1rem; }
  @media (max-width: 767.98px) { .nav { top: .75rem; gap: .625rem; padding: .5rem .875rem; } }
</style>
```

`src/components/Footer.astro`：含 ICP 悬挂
```astro
---
import { site } from '../config/site';
---
<footer class="footer">
  <p>© {new Date().getFullYear()} {site.author} · {site.displayName}</p>
  {site.icp && (
    <p>
      <a href="https://beian.miit.gov.cn/" target="_blank" rel="noopener noreferrer">{site.icp}</a>
    </p>
  )}
</footer>
<style>
  .footer { padding: 4rem 1rem 3rem; text-align: center; color: var(--glaze-ink-soft); font-size: .875rem; }
</style>
```

- [ ] **Step 8: 修改 `Base.astro` 接入 Nav/Footer**

```astro
---
import '../styles/tokens.css';
import '../styles/global.css';
import '../styles/glass.css';
import '../styles/mica.css';
import AuroraBackground from '../components/AuroraBackground.astro';
import Nav from '../components/Nav.astro';
import Footer from '../components/Footer.astro';

export interface Props { title: string; lang?: 'zh' | 'en' }
const { title, lang = 'zh' } = Astro.props;
---
<!doctype html>
<html lang={lang} data-theme="light" data-motion="low">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>{title}</title>
  <link rel="canonical" href="https://blog.lancycelestia.com/" />
  <script is:inline>/* …与 Task 4 Step 5 完全相同，勿改动… */</script>
</head>
<body>
  <AuroraBackground />
  <Nav lang={lang} />
  <main><slot /></main>
  <Footer />
</body>
</html>
```

- [ ] **Step 9: 写主页与全部页面**

`src/pages/index.astro`（Bento，卡片用 **MicaCard 而非 GlassPanel**）：
```astro
---
import Base from '../layouts/Base.astro';
import MicaCard from '../components/MicaCard.astro';
import { getCollection } from 'astro:content';
import { t } from '../i18n/ui';

const posts = (await getCollection('blog', ({ data }) => data.lang === 'zh' && !data.draft))
  .sort((a, b) => b.data.date.valueOf() - a.data.date.valueOf())
  .slice(0, 5);
---
<Base title="澜汐斋记 · 潮声茶馆">
  <div class="bento">
    <section class="hero">
      <p class="eyebrow">潮声茶馆</p>
      <h1>澜汐</h1>
      <p class="sub">霞月 · LancyCelestia</p>
    </section>

    <MicaCard className="cell span-6">
      <h2>{t('latest', 'zh')}</h2>
      <ul>{posts.map((p) => (
        <li><a href={`/blog/${p.data.slug}/`}>{p.data.title}</a></li>
      ))}</ul>
    </MicaCard>

    <MicaCard className="cell span-3">
      <h2>{t('projects', 'zh')}</h2>
      <p><a href="/projects/">chatbot · QQ 聊天机器人</a></p>
    </MicaCard>

    <MicaCard className="cell span-3">
      <h2>{t('about', 'zh')}</h2>
      <p><a href="/about/">澜汐</a></p>
    </MicaCard>

    <MicaCard className="cell span-12">
      <h2>{t('archive', 'zh')}</h2>
      <p><a href="/blog/">全部文章</a> · <a href="/tags/">标签云</a></p>
    </MicaCard>
  </div>
</Base>

<style>
  .bento {
    display: grid; grid-template-columns: repeat(12, 1fr);
    gap: 1.5rem; max-width: 1200px; margin-inline: auto;
    padding: clamp(6rem, 12vw, 9rem) clamp(1rem, 4vw, 4rem) clamp(2rem, 6vw, 5rem);
  }
  .hero { grid-column: span 12; min-height: 38dvh; display: grid; align-content: center; }
  .eyebrow { font-size: .75rem; letter-spacing: .2em; text-transform: uppercase; color: var(--glaze-ink-soft); }
  h1 { font-size: var(--text-h1); margin: .5rem 0; }
  .sub { color: var(--glaze-ink-soft); }
  .span-6 { grid-column: span 6; }
  .span-3 { grid-column: span 3; }
  .span-12 { grid-column: span 12; }
  @media (max-width: 767.98px) {
    .bento { grid-template-columns: 1fr; padding-inline: 1rem; }
    .hero, .span-6, .span-3, .span-12 { grid-column: span 1; }
    .hero { min-height: auto; }
  }
</style>
```

其余页面（about / links / projects / blog index / tags / 404）**均用 `<Base>` 包裹**，`about.astro` 写个人简介 + 项目简介（chatbot QQ 机器人），`links.astro` 先放空态文案。**不得裸写 `<html>`**（v1 的 `submit.astro` 就犯了这个错）。

`src/pages/blog/[...slug].astro` 用 `getStaticPaths` 从 `slug` 派生：
```astro
---
export async function getStaticPaths() {
  const { getCollection } = await import('astro:content');
  const posts = await getCollection('blog', ({ data }) => data.lang === 'zh' && !data.draft);
  return posts.map((p) => ({ params: { slug: p.data.slug }, props: { entry: p } }));
}
const { entry } = Astro.props;
const { Content } = await entry.render();
---
```

- [ ] **Step 10: 写 Playwright 无 JS 测试 `tests/e2e/nojs.spec.ts`**

```ts
import { test, expect } from '@playwright/test';

test.use({ javaScriptEnabled: false });

test('无 JS 时正文与导航仍在', async ({ page }) => {
  await page.goto('/');
  await expect(page.locator('h1')).toContainText('澜汐');
  await expect(page.locator('nav')).toBeVisible();
  await expect(page.locator('footer')).toBeVisible();
});
```

- [ ] **Step 11: 运行确认通过**

Run: `pnpm build && node scripts/assert/content.mjs && pnpm exec playwright test tests/e2e/nojs.spec.ts`
Expected: `CONTENT OK` + 1 passed

- [ ] **Step 12: 故意破坏验证（T2）**

删除 `draft: true` 的过滤条件（改为不过滤）：
Run: `pnpm build && node scripts/assert/content.mjs`
Expected: `draft post leaked to homepage`，退出码 1。**改回。**

- [ ] **Step 13: Commit**

```bash
git add src/content.config.ts src/content src/config src/i18n src/components/Nav.astro src/components/Footer.astro src/layouts/Base.astro src/pages scripts/assert/content.mjs tests/e2e/nojs.spec.ts
git commit -m "feat(content): collections with slug, bento homepage, all pages, no-404 links"
```

---

