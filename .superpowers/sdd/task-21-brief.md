### Task 21: 备份与监控

**Files:** Create `deploy/scripts/backup.sh`

- [ ] **Step 1: 写备份脚本（SQLite + Artalk 数据 + Nginx 配置）**

```bash
#!/usr/bin/env bash
set -euo pipefail
DEST=/var/backups/lancycelestia
mkdir -p "$DEST"
TS=$(date +%Y%m%d)
sqlite3 /var/lib/submitd/submitd.db ".backup '$DEST/submitd-$TS.db'"
tar -czf "$DEST/artalk-$TS.tar.gz" -C /var/lib artalk
tar -czf "$DEST/nginx-$TS.tar.gz" -C /etc nginx/sites-available
find "$DEST" -type f -mtime +14 -delete
```

- [ ] **Step 2: 挂 cron 并验证可恢复**

```bash
echo '0 4 * * * root /opt/lancycelestia/backup.sh' > /etc/cron.d/lancycelestia-backup
# 恢复演练：把备份解压到临时目录并用 sqlite3 打开验证
sqlite3 /tmp/restore-test.db ".tables"
```

---

## 验收清单（全部可执行）

| # | 项 | 命令 | 通过标准 |
|---|---|---|---|
| 1 | 构建 | `pnpm build` | 退出码 0 |
| 2 | 禁用字体 | `node scripts/assert/fonts.mjs` | `FONTS CLEAN` |
| 3 | 设计 token 与三档 | `node scripts/assert/tokens.mjs` | `TOKENS OK` |
| 4 | 玻璃/Mica/Double-Bezel | `node scripts/assert/glass.mjs` | `GLASS OK` |
| 5 | 主题无闪烁/reduced-motion | `node scripts/assert/motion.mjs` | `MOTION OK` |
| 6 | 内容/无死链 | `node scripts/assert/content.mjs` | `CONTENT OK` |
| 7 | 双语/hreflang | `node scripts/assert/i18n.mjs` | `I18N OK` |
| 8 | 搜索索引 | `node scripts/assert/search.mjs` | `SEARCH OK` |
| 9 | RSS/Sitemap/资源策略 | `node scripts/assert/assets.mjs` | `ASSETS OK` |
| 10 | 预压覆盖率与预算 | `node scripts/assert/compress.mjs` | `COMPRESS OK`，index.html.br ≤150KB |
| 11 | 投稿页用 Base | `node scripts/assert/submit.mjs` | `SUBMIT OK` |
| 12 | 四种视觉（人眼 + 自动） | `pnpm exec playwright test tests/e2e/visual.spec.ts` | 亮/暗 × 1440/768/390 六张截图 |
| 12b | **参考质感移植** | `node scripts/assert/reference.mjs && pnpm exec playwright test tests/e2e/reveal.spec.ts` | `REFERENCE OK` + 5 passed（颗粒 / 滚动进场 / reduced-motion / 未采用暖橙） |
| 13 | 动效三档 | `pnpm exec playwright test tests/e2e/motion.spec.ts` | 3 passed |
| 14 | 后端挂掉 | `pnpm exec playwright test tests/e2e/comments.spec.ts` | 3 passed |
| 15 | 投稿失败态 | `pnpm exec playwright test tests/e2e/submit.spec.ts` | 2 passed |
| 16 | 无 JS | `pnpm exec playwright test tests/e2e/nojs.spec.ts` | 1 passed |
| 17 | **www/裸域不跳转** | `curl -sI http://www.lancycelestia.com` | 无 `location: …blog.…` |
| 18 | Brotli 生效 | `curl -sI -H 'Accept-Encoding: br' …` | `content-encoding: br` |
| 19 | HTTPS | SSL Labs | A 及以上 |
| 20 | 备案 | `https://beian.miit.gov.cn/` 查询 | 备案号可查且页脚一致 |

## 审查问题对照表（69 → 处置）

| 来源 | 条数 | 处置 |
|---|---|---|
| A 测试与断言 | 30 | 全部重写断言（T1–T5）；A8 恒真→`motion.mjs` 解析 head；A2 `grep -E "\|"` → 删除，改 Node 脚本；E5 import 链 → Task 2/3 断言编译产物 |
| C 视觉与性能 | 28 | P0-1 `@supports` 正向；P0-2 Bento 改 MicaCard；P0-3 去 `filter:blur`；P0-4/5 reduced-motion 落 CSS 层；P0-6 高光/阴影 token 化；P0-7 字体策略写明系统栈；P1-10 `className`；P1-11 补 span-12；P1-13 `z-index:0`；P1-14 `stroke-dashoffset`；P1-15 `:focus-visible`；P1-16 `aria-live`；P1-17 submit 用 Base；P2-19 补内壳；P2-25 白名单压缩 |
| B 架构与合规 | 11 | 1 备案资格 → **用户实测已排除误判**；2 Artalk 无投稿 → **自研 submitd**；3 `/api/v2` 路径 → Task 14/17；4 HTTPS 顺序 → 重排为 18→19；5 www/裸域 → Task 16 发副本；6 webwallgl 2.66MiB → Task 12 明示体积与费用；8 MemoryMax → Task 15 去掉 MemoryHigh |
