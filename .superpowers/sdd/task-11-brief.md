### Task 11: 文本白名单预压（修正 v1 的「压一切」错误）

**Files:**
- Create: `scripts/compress.mjs`, `scripts/assert/compress.mjs`
- Modify: `package.json`

> **v1 教训**：v1 对 `dist` 全部文件做 Brotli q11，包括已压缩的 AVIF/WOFF2/Pagefind 索引 → `.br` 反而**变大**，而 Nginx 会优先发它，**流量反而增加**。B 审查明确点名此条。

- [ ] **Step 1: 写失败测试 `scripts/assert/compress.mjs`**

```js
import { readdirSync, statSync, existsSync } from 'node:fs';
import { join, extname } from 'node:path';
const fail = (m) => { console.error(m); process.exit(1); };

const TEXT = new Set(['.html', '.css', '.js', '.mjs', '.svg', '.xml', '.json', '.txt', '.webmanifest']);
const SKIP = new Set(['.avif', '.webp', '.woff2', '.png', '.jpg', '.jpeg', '.gif', '.ico', '.pf_fragment', '.pf_meta']);

const walk = (d) => readdirSync(d).flatMap((n) => {
  const p = join(d, n);
  if (/.+\.(br|gz)$/.test(n)) return [];
  return statSync(p).isDirectory() ? walk(p) : [p];
});

const files = walk('dist');
let checked = 0, brTotal = 0, origTotal = 0;
for (const f of files) {
  const ext = extname(f).toLowerCase();
  if (SKIP.has(ext)) {
    // 反向断言：已压缩格式不得生成 .br
    if (existsSync(f + '.br')) fail(`SHOULD NOT precompress binary: ${f}.br`);
    continue;
  }
  if (!TEXT.has(ext)) continue;
  if (!existsSync(f + '.br')) fail(`MISSING ${f}.br`);
  if (!existsSync(f + '.gz')) fail(`MISSING ${f}.gz`);
  const o = statSync(f).size;
  const b = statSync(f + '.br').size;
  if (b >= o) fail(`.br is NOT smaller: ${f} (${o} → ${b})`);
  checked++; brTotal += b; origTotal += o;
}

// 首屏预算（全局断言，v1 缺失）
const idx = 'dist/index.html';
if (existsSync(idx)) {
  const iBr = statSync(idx + '.br').size;
  if (iBr > 150 * 1024) fail(`index.html.br exceeds 150KB budget: ${iBr} bytes`);
}

console.log(`COMPRESS OK (${checked} files, ${origTotal} → ${brTotal} bytes, ${(brTotal/origTotal*100).toFixed(1)}%)`);
```

- [ ] **Step 2: 运行确认失败**

Run: `node scripts/assert/compress.mjs`
Expected: `MISSING dist/index.html.br`

- [ ] **Step 3: 写 `scripts/compress.mjs`（白名单）**

```js
import { readdirSync, statSync, readFileSync, writeFileSync } from 'node:fs';
import { join, extname } from 'node:path';
import { brotliCompressSync, gzipSync, constants } from 'node:zlib';

const TEXT = new Set(['.html', '.css', '.js', '.mjs', '.svg', '.xml', '.json', '.txt', '.webmanifest']);
const MIN_SIZE = 256; // 太小的文件压了反而更大

const walk = (d) => readdirSync(d).flatMap((n) => {
  const p = join(d, n);
  return statSync(p).isDirectory() ? walk(p) : [p];
});

let n = 0;
for (const f of walk('dist')) {
  if (!TEXT.has(extname(f).toLowerCase())) continue;
  const size = statSync(f).size;
  if (size < MIN_SIZE) continue;
  const buf = readFileSync(f);
  const br = brotliCompressSync(buf, {
    params: {
      [constants.BROTLI_PARAM_QUALITY]: 11,
      [constants.BROTLI_PARAM_SIZE_HINT]: buf.length,
    },
  });
  if (br.length < buf.length) { writeFileSync(f + '.br', br); n++; }
  const gz = gzipSync(buf, { level: 9 });
  if (gz.length < buf.length) writeFileSync(f + '.gz', gz);
}
console.log(`compressed ${n} files`);
```

- [ ] **Step 4: 运行确认通过**

Run: `pnpm build && pnpm compress && node scripts/assert/compress.mjs`
Expected: `COMPRESS OK (…)`

- [ ] **Step 5: 故意破坏验证（T2）**

把 `TEXT` 里加入 `'.avif'`：
Run: `pnpm build && pnpm compress && node scripts/assert/compress.mjs`
Expected: `SHOULD NOT precompress binary`（若有 avif 文件）或覆盖率断言失败。**改回。**

- [ ] **Step 6: Commit**

```bash
git add scripts/compress.mjs scripts/assert/compress.mjs package.json
git commit -m "feat(build): brotli/gzip precompression limited to text whitelist"
```

---

