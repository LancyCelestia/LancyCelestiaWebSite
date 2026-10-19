# 参考站点技法提炼：daniya.top

> 采集方式：CDP 浏览器实测（DOM / computedStyle / CSSOM / main.js 源码）+ 整屏截图
> 采集时间：2026-10-20
> 截图：`docs/reference-daniya-hero.png`

---

## 一、实测到的视觉参数

| 项 | 实测值 |
|---|---|
| 页面底色 | `#05070d` / `#080b14` / `#0b101c`（三层递进） |
| 正文色 | `#eef3f9`，次级 `#a9b5c8`、`#6d7a90`、`#48536b` |
| 主色三支 | `--void #8ed9ec`（青）、`--violet #b9a8ff`（紫）、`--ember #ffb98a`（暖橙） |
| 虹彩渐变 | `linear-gradient(100deg,#8ed9ec 0%,#b9a8ff 46%,#ffb98a 100%)` |
| 发丝线 | `rgba(255,255,255,.08)` / `.16` |
| 显示字体 | `Cormorant Garamond, "Times New Roman", "Songti SC", serif` |
| 正文字体 | `Noto Sans SC, -apple-system, "PingFang SC", "Microsoft YaHei", sans-serif` |
| 行高 | **1.85** |
| 页面总高 | 7387 px |
| 资源 | 1 个 canvas、12 个 SVG、**0 张位图**、JS 仅 **13 KB** |

---

## 二、实测到的 8 个动画（CSSOM `@keyframes` 原文）

| # | 名称 | 触发 | 实测实现 |
|---|---|---|---|
| 1 | `grain-shift` | 常驻 | `.grain{position:fixed;inset:-50%;z-index:60;opacity:.05;background-image:url("data:image/svg+xml,…feTurbulence type='fractalNoise' baseFrequency='.9' numOctaves='2' stitchTiles='stitch'…");animation:9s steps(6) infinite grain-shift}` |
| 2 | `ring-spin` | 常驻 | `.hero__ring{position:absolute;top:50%;left:50%;width:min(92vmin,900px);transform:translate(-50%,-50%);animation:120s linear infinite ring-spin}` |
| 3 | `panel-in` | **滚动** | `0%{opacity:0;transform:translateY(16px)} 100%{opacity:1;transform:translateY(0)}`；**17 个元素**用 `IntersectionObserver` 触发 |
| 4 | `float-y` | 常驻 | `0%{translateY(-6px)} 100%{translateY(7px)}` |
| 5 | `pl-float` | 常驻 | `0%,100%{translateY(0)} 50%{translateY(-7px)}` |
| 6 | `pl-slide` | 局部 | `0%{translateX(-110%)} 100%{translateX(360%)}`（光带扫过） |
| 7 | `scroll-drop` | 首屏底部 | `translateY` + `opacity` 循环 |
| 8 | `pl-fade` | 常驻 | `100%{opacity:1}` |

**Canvas 英雄区**：`<canvas class="hero__canvas" width=2310 height=1309>`，JS 用 `arc()` + `createRadialGradient()` 绘制**泡泡**，`requestAnimationFrame` 驱动；`main.js` 共 13 KB。

---

## 三、质感法则（从截图读出，比代码更重要）

1. **大面积纯黑留白**：视觉重量集中在中心一个发光圆环，四周全是黑
2. **极细字重**：标题用 Cormorant 的细体，大字靠**体量**压场，不靠粗体
3. **光只在中心晕开**：没有满屏渐变，边缘纯黑
4. **所有动效都极慢**：120s 旋转、9s 颗粒、7px 漂浮——这是高级感的真正来源
5. **无卡片投影**：层次靠「发光 / 不发光」区分，不靠阴影
6. **大标题 + 极小字号的辅助信息**（SCROLL、竖排「星炬学院·虚质科学部」）形成强烈对比

---

## 四、本项目采用范围（用户决策：B = 1 + 3）

**采用**：
- 质感法则 **1–6 全部**（这是「优点」的主体）
- 特效 **#1 全局胶片颗粒**
- 特效 **#3 滚动进场（IntersectionObserver + panel-in）**

**不采用**（用户明确排除）：
- #2 泡泡 canvas（与「潮声茶馆」意象不符）
- #4/#5 漂浮位移
- #6 光带扫过
- #7 滚动提示
- #8 pl-fade

**色板替换**（主体风格不变）：
- 保留 daniya 的「近黑深底 + 极细字重 + 中心发光 + 超慢动效」结构
- 颜色用本项目的青—蓝—紫 + 粉（`--glaze-celadon / --glaze-abyss / --glaze-orchid / --glaze-coral`）
- **不使用暖橙 `#ffb98a`**（与青瓷冲突）

**主页展现方式借鉴**：
- 首屏改为「**中心发光 + 巨型标题 + 极小辅助字**」的 Hero 结构
- 潮线（TideLine）承担 daniya「圆环」的角色
- 章节标题带极小字号 eyebrow（如 `PROFILE` / `ABILITY` 那种）

---

## 五、三档动效分配（用户已确认）

| 档位 | 颗粒 | 滚动进场 | Aurora | 潮线 |
|---|---|---|---|---|
| high | ✓ 9s steps(6) | ✓ 800ms | ✓ 3 层 | ✓ |
| mid | ✓ 但 `animation: none`（静态噪点保留） | ✓ 400ms | ✓ 2 层 | ✓ |
| low | ✗ 完全不渲染 | ✗ 直接可见 | ✗ 静止 | ✗ 静止 |

`prefers-reduced-motion: reduce` → 等同 low，且**在 CSS 层强制**（不依赖 JS）。
