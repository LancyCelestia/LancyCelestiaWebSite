# 潮声茶馆 v1 设计规格

> **状态**：待审阅（v0.9，事实已核实，等你确认 5 个待决项）
> **日期**：2026-10-20
> **作者**：DSH 编程模式（Lead）
> **审阅人**：澜汐
> **站点**：https://www.lancycelestia.com
> **配套文档**：[事实核查台账](./2026-10-20-fact-check-ledger.md)（所有数字的出处与置信度）

---

## 0. 一句话

一个以**纯静态产物分发**为核心、**全部视觉动效由客户端 GPU 渲染**、**服务器只负责发字节**的中英双语个人博客。视觉是一杯「清透青瓷 × 潮声」的液态玻璃，主色取自守岸人的深海蓝、爱弥斯的粉、达妮娅的粉紫。

**核心设计原则**：ECS 出网 **0.8 元/GB** 是全局最贵的资源（台账 2.1），所有架构决策都服从「把字节搬离 ECS」这一条。

---

## 1. 已确认基线

| 维度 | 确认内容 | 来源 |
|---|---|---|
| 备案主体 | 个人，浙江杭州（与 ECS 同省，最顺情形） | A1/A2 |
| 现有备案 | 无，首次新增备案 | A3 |
| 备案操作人 | 你本人（短信核验 + 人脸核身） | A6 |
| 服务器 | Ubuntu 26.04 / 2 vCPU / 2 GiB / ESSD 40 GiB / 杭州 / **按流量计费** / 100 Mbps 峰值（非保底） | 图1 |
| SSH | 密钥登录，保留 22 端口，机器上无其他服务 | C1/C3/C4 |
| 域名 | `lancycelestia.com`，主访问域 `www.`，阿里云万网已实名，DNS `dns11/12.hichina.com` | 图2 / Q7 |
| 子域 | `chatbot.lancycelestia.com` 本期不规划 | Q7 |
| HTTPS | 强制 HTTPS；非 www 跳 www | B2 |
| 部署 | 本地构建 → 只传产物 → 服务器纯分发 | F3 |
| 代码托管 | GitHub + 本地 | F2 |
| 压缩 | Brotli 拉满 | F5 |
| 备份 | 自动备份 | F4 |
| 语言 | 中英双语 | D6 |
| 名号 | 展示名「潮声茶馆」，主名「澜汐」，副名「霞月」，英文 LancyCelestia | E1 |
| 配色 | 青—蓝—紫 + 粉 | E2 |
| 主题 | 亮暗可切换 | E3 |
| 动效 | 强烈，全部客户端渲染，三档可调 | E6 |
| 降级 | 接受 reduced-motion 与无 backdrop-filter 兜底 | E7 |

---

## 2. 视觉设计规格

> 依据 `high-end-visual-design` 的 Variance Engine：**Vibe = Ethereal Glass（青瓷变体）**，**Layout = Asymmetrical Bento**。
> 刻意**不选** Editorial Luxury（暖奶油 + 衬线）——那是 AI 默认套路，且与「清透青瓷」的冷调冲突。

### 2.1 色彩 Token

| Token | 亮色（浅釉） | 暗色（深釉） | 用途 |
|---|---|---|---|
| `--glaze-ink` | `#0B1622` | `#E8F1F8` | 正文 |
| `--glaze-ink-soft` | `#41566B` | `#9FB3C6` | 次要文字 |
| `--glaze-canvas` | `#F2F7FA` | `#060B12` | 页面基底 |
| `--glaze-porcelain` | `#FBFDFF` | `#0D151F` | 面板内核 |
| `--glaze-abyss` | `#123A5C` | `#071628` | 守岸人深海蓝（主锚） |
| `--glaze-celadon` | `#3E9E8F` | `#5FD6C4` | 青瓷青绿（点睛） |
| `--glaze-coral` | `#F2729B` | `#FF8FB4` | 爱弥斯粉（情绪） |
| `--glaze-orchid` | `#9B6BD6` | `#B58CF2` | 达妮娅粉紫（过渡） |
| `--glaze-hairline` | `rgba(11,22,34,.08)` | `rgba(232,241,248,.10)` | 发丝描边 |

主渐变 `--glaze-aurora`：
```
亮: linear-gradient(135deg,#3E9E8F 0%,#6FA8D8 32%,#9B6BD6 66%,#F2729B 100%)
暗: linear-gradient(135deg,#5FD6C4 0%,#4E8FD0 32%,#B58CF2 66%,#FF8FB4 100%)
```

### 2.2 四种必备视觉风格的落地定义

#### ① 动态釉瑚渐变色

**不是**静止渐变。釉的特征是窑变——颜色随时间与视角流动。

- `position:fixed; inset:-20%; z-index:-1` 的 `.aurora`，内部叠 **3 层** `radial-gradient` 色斑（青绿/粉/粉紫），各自 `animation-duration` 28s / 37s / 45s，做 `translate` + `scale` 循环。
- 再叠一层 `conic-gradient`，以 `hue-rotate` 60s 极缓旋转，制造「釉面流动」。
- **只用 `transform` 与 `opacity`**，全走 GPU 合成层，主线程零开销。
- 亮色主题低透明度（0.35–0.5）叠瓷白 = 薄釉；暗色高饱和叠深底 = 厚釉。

#### ② 液态玻璃

三层结构：
1. **折射层**：`backdrop-filter: blur(24px) saturate(180%) contrast(1.05)`
2. **高光层**：`::before` 用 `linear-gradient(140deg, rgba(255,255,255,.5), transparent 40%)` + `mix-blend-mode:screen`，模拟上缘镜面反光
3. **液面层**：`::after` 的 `radial-gradient` 高光跟随指针（CSS 变量 `--mx/--my` 由极少 JS 更新）

> ⚠️ **硬护栏**：`backdrop-filter` **只用于固定/吸顶元素与小型面板**。绝不用于整页滚动容器——会导致移动端持续重绘、严重掉帧。

#### ③ 面板背景半模糊

统一封装 `.panel`，强度由 CSS 变量控制，与动效档位联动：

| 档位 | `--panel-blur` | `--panel-sat` | 说明 |
|---|---|---|---|
| high | `24px` | `180%` | 完整液态玻璃 |
| mid | `14px` | `150%` | 保留玻璃感 |
| low | `0px`（改 92% 不透明纯色） | — | 弱设备/省电 |

#### ④ Mica 云母质感

Mica 的核心 = **从背景取色 + 噪点纹理 + 极低频漫射**。

- **取色层**：面板底色不是纯色，用 `color-mix(in oklab, var(--glaze-abyss) 8%, var(--glaze-porcelain) 92%)` 从主色「渗」一点点，避免死白/死黑。
- **噪点层**：内联 SVG `feTurbulence` 转 data-URI（**约 300 字节，零图片请求**），`opacity:.035`，`mix-blend-mode:overlay` —— 这是 Mica 的指纹。
- **漫射层**：`box-shadow: 0 24px 80px -32px rgba(7,22,40,.45), inset 0 1px 0 rgba(255,255,255,.14)`
- **Double-Bezel 嵌套**：外层 `p-1.5` 半透壳 + 发丝环，内层自有底色与内高光，圆角共心 `calc(var(--r) - 6px)`。

### 2.3 排版（含 CLS 硬约束）

- **Display**：拉丁用 **Fraunces**（可变衬线，光学尺寸轴）。刻意避开 Inter/Roboto/Helvetica。中文标题回退 `"Songti SC","Source Han Serif SC",serif`。
- **Body**：拉丁 **Sora**；中文**走系统字体栈**（`-apple-system,"PingFang SC","Microsoft YaHei",sans-serif`）。
- **Mono**：`ui-monospace,"JetBrains Mono",Consolas,monospace`（代码块与「终端」气质）。
- **字阶**：H1 `clamp(2.6rem,6vw,4.5rem)`；H2 `clamp(1.8rem,3.4vw,2.6rem)`；正文 `clamp(1rem,1.05vw,1.0625rem)`，行高 1.75（中文需更松）。

> **为什么中文倾向不用 Web 字体（已按实测下调论断强度）**：
> 实测数据：Noto Sans SC 单字重**全量** WOFF2 = **1.09 MiB**（台账 5.3），**不是** 4–10 MB；走 unicode-range 分片后，一个中文页**实际只命中 3–6 片 ≈ 30–150 KB**（台账 5.3b）。
>
> 因此这是一个**取舍**而非「硬性必须」：
> - 选**系统字体栈**：每页省 30–150 KB，零 CLS，跨平台字形不一致。
> - 选**Web 字体（分片加载）**：每页多 30–150 KB ≈ 按 0.8 元/GB 折算约 **0.024–0.12 元/千次浏览**，换来品牌一致性。
>
> **默认决策：系统字体栈**（符合「极致省流量」的首要目标）。若你更看重排版一致性，可在 D6 改选「分片 Web 字体」——成本可算，不致命。
>
> **字体加载硬约束（若启用拉丁 Web 字体）**：仅 `latin` 子集，Display 1 字重 + Body 2 字重，**合计 ≤ 90 KB WOFF2**；超限则砍回系统栈。
>
> **禁止 `font-display: swap`**（台账 5.5：web.dev 明确警告会加剧 CLS）。用 `font-display: optional` + `size-adjust`/`ascent-override` 做度量对齐（台账 5.6）。

### 2.4 布局（Asymmetrical Bento）

```
┌──────────────────────────────────────────────────────┐
│  (浮动胶囊导航 mt-6, 毛玻璃)  ◦ ◦ ◦                   │
├──────────────────────────────────────────────────────┤
│  ┌─ Eyebrow: 潮声茶馆 ─────────────────────────┐      │
│  │  澜汐           ← 巨型 Display               │      │
│  │  霞月 · LancyCelestia                        │      │
│  │  [ 头像 128px 圆 ]  [ 进入茶馆 ↗ ]           │      │
│  └─────────────────────────────────────────────┘      │
│                                                      │
│  ┌────────────┬──────────┬──────────┐                 │
│  │ 最新文章    │ 项目     │ 关于我    │  ← Bento       │
│  │ span-6     │ span-3   │ span-3   │                 │
│  ├────────────┴──────────┴──────────┤                 │
│  │ 标签云 / 归档时间线 (span-12)      │                 │
│  └───────────────────────────────────┘                 │
└──────────────────────────────────────────────────────┘
```
- <768px：全部塌陷单列，`px-4`，去位移与旋转，`gap-6`。
- 不用 `h-screen`，统一 `min-h-[100dvh]`（防 iOS 地址栏抖动）。

### 2.5 动效三档

`<html data-motion="low|mid|high">`，CSS 全量用变量响应：

| 动作 | low | mid | high |
|---|---|---|---|
| 釉瑚渐变 | 静态 | 2 层 30s | 3 层 28/37/45s + conic |
| 面板模糊 | 0px | 14px | 24px |
| 指针高光 | 关 | 关 | 开 |
| 滚动进场 | 关 | fade-up 400ms | fade-up + blur 800ms |
| 页面转场 | 关 | 关 | ClientRouter |
| 蝴蝶/粒子 | 关 | 关 | 开（Canvas，可关） |
| 缓动 | — | `cubic-bezier(.32,.72,0,1)` | 同 + 1200ms |

**自动降档**（首帧内完成，防闪烁）：
1. `prefers-reduced-motion: reduce` → 强制 `low`
2. `hardwareConcurrency ≤ 4` 或 `deviceMemory ≤ 4` → 默认 `mid`
3. 不支持 `CSS.supports('backdrop-filter','blur(1px)')` → 强制 `low`
4. **用户手动选择存 localStorage，优先级最高**

### 2.6 签名元素：潮线（Tide Line）

一条贯穿页面、随时间缓慢起伏的极细渐变线（SVG path + `stroke-dasharray` 动画），串起导航下方、章节之间、页脚。呼应「潮声」，是釉面流动的可视化，成本极低（1.5px 的 SVG）。

其余保持克制：不加插画墙、不加图标堆砌。

### 2.7 无障碍与降级（硬性）

- 正文对比度 ≥ 4.5:1（亮暗双主题都过）
- 键盘焦点：`outline:2px solid var(--glaze-celadon); outline-offset:3px`
- 装饰动效层 `aria-hidden="true"` + `pointer-events:none`
- 无 `backdrop-filter` 时 `.panel` 回落 92% 不透明纯色（可读性优先）

---

## 3. 技术架构

### 3.1 选型（全部有台账支撑）

| 层 | 选择 | 理由（台账） |
|---|---|---|
| 站点框架 | **Astro 7.4.1** | 默认剥离客户端 JS（4.2）；构建期静态产物（4.6）；内置 i18n（4.4）；内置 sharp 图片优化（4.3） |
| 运行时要求 | Node ≥ 22.22.0 | 4.1；本机 v26.7.0 ✓（4.14） |
| 样式 | 原生 CSS + CSS 变量（**不引 Tailwind**） | 产物最小；玻璃/Mica 靠原生能力，无需框架 |
| 交互 | 原生 TS + 少量 `<script>`，无 UI 框架 | 零框架运行时 |
| 服务器 | **Nginx**，只发静态文件 | 零渲染、零 SSR |
| 压缩 | `brotli_static` + `gzip_static` 双预压 | apt 可装无需编译（4.8）；可共存协商（4.10） |
| HTTPS | **Let's Encrypt + certbot** | 阿里云证书 200 天需手动续（6.1）；LE 90 天自动续可绕开 |
| 图片 | 构建期 Astro 生成 AVIF/WebP 多尺寸 | 4.3；不依赖外部图床 |
| 评论 | giscus 或自建，**可切换** | 你要求 A+B 可开关 |
| 部署传输 | **本地构建 → 产物同步 → Nginx 重载** | 你指定 F3 |

> **传输工具的确定（解决台账 4.15「本机无 rsync」）**：
> 本机无 rsync，Windows 原生不可用。**已核实的关键约束**：
> - `scp` / `sftp` **都没有增量同步能力**与（OpenSSH 9.0 起 scp 默认走 SFTP 协议，仅 `-r` 递归，`-a` 只是单文件断点续传，不是增量）。
> - **关键陷阱**：即使服务器 `apt install rsync` 也无用——**rsync 要求客户端与服务端两端都有 rsync 二进制**。本机没有，链就断。
>
> **三种可行路径**（按推荐度排序）：
> 1. **推荐：本地装 rsync（通过 WSL 或 msys2），服务器装 rsync** → 真正的增量同步 `rsync -az --delete`。一次配置，长期最优，图片增长后优势明显。
> 2. **零安装兜底：本地 `tar -czf` 打包 dist → `scp` 传单个压缩包 → 服务器解包**。只有一次传输、服务器仅需 tar/gzip（系统自带），**首版最省事**。
> 3. 不推荐：产物提交到 Git 分支 —— 污染历史与体积，除非用 orphan 分支/独立仓库。
>
> **决策**：P5 用 **路径 2（tar+scp）跑通首版，零新增依赖**；P8 升级到 **路径 1（rsync 增量）**。首版不阻塞，长期不欠债。
>
> **零停机发布协议**（P5 起即采用）：
> ```
> 1. 本地: tar -czf dist.tar.gz -C dist .
> 2. scp dist.tar.gz user@host:/tmp/
> 3. 服务器: mkdir /var/www/lancycelestia/releases/<时间戳>
>            tar -xzf /tmp/dist.tar.gz -C releases/<时间戳>
> 4. 服务器: ln -sfn releases/<时间戳> /var/www/lancycelestia/current   ← 原子切换
> 5. 保留最近 5 个 release，其余删除
> ```
> Nginx `root` 指向 `current`（软链）。`ln -sfn` 是原子操作，切换瞬间无 404/半截页面。
> 注意：Nginx 默认 `disable_symlinks off`，软链可用；但**软链的每级父目录需对 nginx 运行用户可读**。

### 3.2 为什么不用 Next.js / Nuxt

你要「性能最好、压缩最好」。Next/Nuxt 的 SSR 模式需要 Node 常驻进程（100–200 MB+），直接撞 2 GiB 红线（图1 已提示 OOM 风险）。Astro 的静态产物由 Nginx 直接发，**服务器内存占用可压到 < 50 MB**。

### 3.3 「注册/投稿/评论」的架构处理

你要求 **A+B 一次做 + 所有功能可开关**。方案：

- **默认形态（A）**：纯静态 + giscus（评论）+ GitHub PR（投稿）。**服务器零负载**。
- **增强形态（B）**：Go 单二进制 + SQLite，只挂在 `/api/*` 与 `/auth/*`，Nginx 反代。常驻约 30–60 MB。
- **开关机制**：`site.config.ts` 里 `features: { comments:'giscus'|'self'|'off', submissions:boolean, auth:boolean }`，构建期决定是否加载对应路由与脚本。
- **不可同时开两套评论**（数据会分裂）——这是约束，需你知晓。
- **注**：你 Q1 答「A+B 一次做」，与 D5 的「预留接口」是**同一件事的两种表述**。本文档的最终口径以 **D5=C（一期 A 形态 + 预留 `/api` 边界）** 为准；若你要**一期就把 B 全量做完**，请直接说「D5=B」，我会把 Go 服务纳入 P3，代价是工期增加约 30%，且必须做 swap 兜底。

### 3.4 目录结构

```
lancycelestia.com/
├─ docs/superpowers/specs/        # 本文档与台账
├─ src/
│  ├─ styles/
│  │  ├─ tokens.css               # 色彩/间距/动效变量
│  │  ├─ glaze.css                # 动态釉瑚渐变
│  │  ├─ glass.css                # 液态玻璃 + 半模糊
│  │  └─ mica.css                 # 云母质感 + 噪点
│  ├─ components/
│  │  ├─ Nav.astro  Hero.astro  Panel.astro
│  │  ├─ TideLine.astro           # 签名元素
│  │  ├─ MotionSwitch.astro       # 三档切换
│  │  └─ Wallpaper.astro          # 动态壁纸入口
│  ├─ layouts/Base.astro
│  ├─ pages/
│  │  ├─ index.astro  posts/  tags/  categories/
│  │  ├─ about/  projects/  links/  search/
│  │  └─ en/…                     # 英文镜像
│  └─ content/{posts,projects}/   # Markdown 内容集合
├─ scripts/
│  ├─ compress.mjs                # 构建后生成 .br/.gz 预压
│  └─ deploy.mjs                  # 产物同步
└─ deploy/nginx/lancycelestia.conf
```

### 3.5 数据流

```
本地写 Markdown → pnpm build（Astro） → dist/ + .br/.gz 预压
   → 同步到服务器 /var/www/lancycelestia/current
   → nginx -t && systemctl reload nginx
```
**服务器不参与构建**（2 vCPU 上跑 Astro 构建会吃满 CPU 数分钟）。

---

## 4. 省流量措施（逐项 + 效果口径）

> **诚实声明**：台账 5.2 已核实，「Brotli 比 gzip 省 15–25%」**无一手权威来源**。下表凡是标「需实测」的，我会在阶段 6 用真实产物跑一次测量，把实测数字回填，**不用业界传说充数**。

| # | 措施 | 预期效果 | 效果性质 |
|---|---|---|---|
| 1 | **中文不用 Web 字体，走系统栈** | 每页省 **30–150 KB**（实测：分片后中文页实际命中 3–6 片） | 台账 5.3b；全量 1.09 MiB |
| 2 | 拉丁字体仅 latin 子集 + 3 字重，≤90 KB | 对比全量 latin 家族（数百 KB – MB） | 需实测 |
| 3 | **构建期 Brotli q11 预压** | 文本体积显著下降（RFC 定性 prior art，量化**需实测**） | 需实测 |
| 4 | `.gz` 兜底（老客户端只发 gzip 时） | 避免退化为未压缩 | 台账 4.10 |
| 5 | 图片**构建期转 AVIF** + 多尺寸 + `srcset` | 相对 JPEG/PNG 通常大幅下降 | 需实测（每张图） |
| 6 | 图片**移出 ECS**（走 OSS/CDN 或独立域） | ECS 出网从 0.8 元/GB 的路径上移走 | 台账 2.1/2.2 |
| 7 | Immutable 长缓存（`max-age=31536000,immutable`）+ 内容哈希文件名 | 回访用户 **0 字节**重复下载 | 确定 |
| 8 | HTML 短缓存 + ETag | 二次访问变 304（仅几百字节） | 确定 |
| 9 | 零客户端 JS（Astro 默认）+ 极少 `<script>` | 首屏 JS 目标 **< 15 KB br** | 需实测 |
| 10 | 内联关键 CSS，其余异步 | 消除阻塞请求 | 需实测 |
| 11 | 内联 SVG 噪点（300 字节）替代背景图片 | 省一次请求 + 数 KB | 确定 |
| 12 | 无第三方字体/统计/图标 CDN | 免去对外部 CDN 的依赖与请求 | 确定 |
| 13 | 关闭目录列表、限速与防爬（`limit_req`）+ robots | 削掉扫描器流量 | 确定 |
| 14 | 可选：CDN 前置（按你 Q2=B） | 把下行挪到 0.24 元/GB | 台账 2.5/2.6 |

**月度估算（推断，非官方）**：假设月 PV 5,000、单页首次 ~120 KB（br 后）、回访率 40%：
`5000 × 0.6 × 120KB ≈ 360 MB/月 ≈ 0.36 GB → 0.8 元/GB ≈ 0.29 元/月`（不含图片）。
> 图片必须外置，否则单张 200 KB 的图被看 1000 次就是 200 MB。

---

## 5. 部署与备案

### 5.1 关键流程修正（重要）

台账 1.5 已确认官方原文「**备案成功后搭建网站**」——**备案不要求网站已可访问**。

因此两条线**并行**：

```
线 A（你操作）：阿里云备案系统提交 → 短信核验 → 人脸核身 → 管局审核（约 1–20 工作日）
线 B（我产出）：本地建站 → 构建 → 预压 → 产物就绪
合并点：备案通过 → DNS 解析生效 → HTTP 可访问 → certbot 签发证书 → 强制 HTTPS
```

### 5.2 阶段划分

| 阶段 | 内容 | 谁执行 | 验收 |
|---|---|---|---|
| P0 | 服务器基线：更新、创建部署用户、防火墙、时间同步 | **你**（我给脚本） | `nginx -V` 输出存档 |
| P1 | Nginx + brotli/gzip_static 模块 + 站点配置骨架 | **你** | `nginx -t` 通过；`curl -I` 返回 200 |
| P2 | 本地项目初始化 + 设计系统落地 | 我（本地写代码） | 构建通过、视觉验收 |
| P3 | 内容与功能模块（含开关） | 我 + 你审 | 各页面可访问 |
| P4 | 构建产物 + 预压 + 体积实测 | 我 | 回填 §4 的「需实测」 |
| P5 | 同步部署 + 首次上机验证 | **你**（我给脚本） | 外网可访问、HSTS |
| P6 | HTTPS（certbot + 自动续期） | **你** | 证书链正确、`--dry-run` 续期成功 |
| P7 | 备案提交与跟踪 | **你** | 备案号到手、页脚悬挂 |
| P8 | 备份 + 监控 + 安全加固 | 我出方案，**你**执行 | 备份可恢复演练 |

### 5.3 备案材料清单（个人 / 浙江）

- 身份证正反面
- 手机号 + 邮箱（短信核验）
- 阿里云 App 做人脸核身
- 网站名称：**按 D1 决策填写**
- 服务器备案服务码（**待补充：你的额度现状**）

### 5.4 备案后的合规悬挂

- 页脚固定显示：`ICP 备案号` + 链接 `https://beian.miit.gov.cn/`
- 若有公安备案号，一并悬挂

---

## 6. 测试与验收

| 项 | 方法 | 通过标准 |
|---|---|---|
| 构建 | `pnpm build` | 退出码 0，无 TS 错误 |
| 视觉 | 桌面 1440 / 平板 768 / 手机 390 三档截图 | 四种视觉风格可见；无布局溢出 |
| 动效三档 | 手动切换 + 系统降级模拟 | 三档差异明显；reduced-motion 生效 |
| 性能 | Lighthouse（移动模拟） | Performance ≥ 90，CLS < 0.1 |
| 体积 | `du -sh dist` + 逐资源 gzip/br 对比 | 首屏首访 HTML+CSS+JS ≤ 150 KB（br） |
| 压缩 | `curl -H 'Accept-Encoding: br' -I` | 返回 `content-encoding: br`，`.br` 命中 |
| HTTPS | SSL Labs | A 级及以上 |
| 可访问 | 外网多网络访问 | 200，无证书警告 |
| 备案 | 工信部查询 | 备案号可查，页脚一致 |

---

## 7. 参考站点技法（daniya.top）与采用范围

> 完整实测记录见 `2026-10-20-reference-daniya-analysis.md`，截图 `docs/reference-daniya-hero.png`。
> 用户决策：**B = 1 + 3** —— 学它的优点与质感，主线风格不变，主页可学其展现方式。

| 采用 | 不采用 |
|---|---|
| **质感法则 1–6**：大面积纯黑留白、极细字重靠体量压场、光只在中心晕开、**所有动效极慢**、无卡片投影靠发光分层、大标题与小字号强对比 | 泡泡 canvas |
| **特效 #1 全局胶片颗粒**（内联 `feTurbulence`，9s `steps(6)`） | 漂浮位移 |
| **特效 #3 滚动进场**（`IntersectionObserver` + `panel-in`，`translateY(16px)`→0） | 光带扫过、滚动提示、`pl-fade` |
| **主页 Hero 结构**：中心发光 + 巨型标题 + 极小 eyebrow | 暖橙 `#ffb98a`（与青瓷冲突） |

**色板不替换**：仍用 `--glaze-celadon / --glaze-abyss / --glaze-orchid / --glaze-coral`。
**潮线承担 daniya「圆环」的角色**（它的视觉锚点，在本项目由 TideLine 扮演）。

**三档分配**（用户已确认）：

| 档位 | 颗粒 | 滚动进场 | Aurora | 潮线 |
|---|---|---|---|---|
| high | ✓ 9s steps(6) | ✓ 800ms | ✓ 3 层 | ✓ |
| mid | 静态噪点（`animation:none`） | ✓ 400ms | ✓ 2 层 | ✓ |
| low | 不渲染 | 直接可见 | 静止 | 静止 |

---

## 8. 现成开源项目复用清单（均已**克隆实读**核实，非仅看 README）

| 用途 | 项目 | License | 核实方式 | 复用方式 |
|---|---|---|---|---|
| 站点骨架 / 玻璃 UI / Aurora / Pagefind / 明暗切换 | `kpab/astro-haze` | **MIT** | 克隆 + 读源码 | 搬 `glass.css`、`AuroraBackground.astro`、`tokens.css`、`SearchModal`、`ThemeToggle`；**替换** Inter 字体、紫色 token、加 i18n |
| WE 壁纸全参数渲染 | `oneincase/webwallgl` v3.0.0 | **MIT** | 克隆 + 读 `types.ts` + npm | 直接集成；属性类型 `bool→boolean / slider→number / combo·textinput·file·directory→string / color→"r g b"` |
| 粒子 / 蝴蝶动效（兜底） | `tsparticles/tsparticles` | **MIT** | 调研核实 8.9k★ | 仅在 webwallgl 不可用时启用 |
| 评论 + 注册登录 + 鉴权 | `ArtalkJS/Artalk` | **MIT** | 调研核实 2.3k★，go.mod 确认 `modernc.org/sqlite`（纯 Go 无 cgo） | 直接用；补一张 posts 表 + 投稿路由 |

**不采用**（已核实原因）：
- `vanta.js`：MIT 但**实质停维护**，参数过少，不满足需求
- `rocksdanister/lively` / `WallpaperMachine`：**GPL**，与 MIT 项目不兼容，且无 Web 版
- `commento`：强制 PostgreSQL、停更 5 年
- `remark42`：BoltDB/Badger 非 SQLite，且过重
- `jsDelivr` / `ImgBB`：大陆不可靠（台账 3.1–3.3）

## 9. ⚠️ 壁纸资源法律约束（必须知晓）

`webwallgl` README 原文大意：

> Wallpaper Engine 工坊资源（`scene.pkg`、贴图、音频、视频）归各自作者所有：**只能使用你拥有或已获授权的资源，不要把别人的作品打包进你的产品，也不要公开再分发。**

**对本项目的约束**：
1. ✅ 可以使用 `webwallgl` **引擎本身**（MIT）
2. ✅ 可以使用：你自己创作的壁纸、作者明确授权/CC 协议的壁纸、引擎自带的简单渐变壁纸
3. ❌ **不可以**把从创意工坊下载的壁纸打包进博客分发给访客

**默认策略**：首版只内置**纯 CSS 釉瑚渐变**作为默认背景（零资源、零风险），`webwallgl` 作为「用户自选壁纸」入口，**不预置任何第三方壁纸资源**。



### 7.0 已拍板（全部确认完毕 · 第三轮修订）

| 编号 | 决策 | 结果 | 备注 |
|---|---|---|---|
| **D1** | **备案网站名称 = 「澜汐斋记」** | ✅ 4 汉字、无禁用词 | |
| D2 | 图片托管 | ✅ 构建期 AVIF，放仓库 | |
| **D3** | 主访问域 = **`blog.lancycelestia.com`** | ✅ | 见 D3-b |
| **D3-b** | **`www` 与裸域：各放一份 blog 页面副本，不做 301** | ✅ 用户明确要求 | 用 `<link rel="canonical">` 指向 `blog.` 解决重复内容，**不是 301** |
| D4 + Q-b | 动态壁纸 | ✅ **两者都做（B3/C）**：CSS 釉瑚渐变默认，`webwallgl` 作为高级可选 | 见 §10 流量约束 |
| **D5** | 注册/登录/评论/投稿 | ✅ **评论+注册登录用 Artalk；投稿自研 Go+SQLite** | 用户选 B（自研） |
| D6 | 中文字体 | ✅ 代码可部署、默认不启用 | |
| D7 | HTTP/3 | ✅ 放弃 | |
| Q-a | 复用策略 | ✅ B+C 混合 | |
| **A** | `www`/裸域处理 | ✅ **复制 blog 页面**（见 D3-b） | |

### 7.1 已解除的阻塞（第三轮核实）

| 原判 | 更正 | 依据 |
|---|---|---|
| ~~实例不在可备案列表~~ | ✅ **在列表内**，且**免费备案服务码 4/5（剩 1 个未用）** | 用户截图：`launch-advisor-20260519 / i-bp1a6bjfudd2u7fqwv1pd / 4-5 / 0` |
| ~~可能需改带宽计费~~ | ✅ **不需要改**，100 Mbps 按流量计费 + 包年包月 + 公网 IP 已满足 | 同上截图 |
| ~~备案服务码额度未知~~ | ✅ **已知**，无需另行购买 | 同上 |

> ⚠️ **注意**：截图显示「免费备案数量（已使用/上限）= 4/5」，即**已用 4 个、仅剩 1 个**。本文档第一次观察时误读为「0/5」。剩余额度足够本项目（1 个域名 = 1 个服务码），但**耗尽了就要购买**。

---

## 10. 风险登记

| 风险 | 影响 | 缓解 |
|---|---|---|
| 2 GiB 内存 OOM（图1 已提示） | 站点不可用 | 零常驻服务；加 2 GB swap 兜底；不开 Docker |
| 「按流量计费」被爬虫打爆 | 费用失控 | `limit_req` + 屏蔽恶意 UA + robots + 可选 CDN |
| 备案名称被驳回 | 上线推迟 | D1 选最保守名；预留改名重提方案 |
| 证书 200 天到期 | HTTPS 中断 | 用 certbot 自动续期，绕开阿里云订阅制 |
| 服务器包未编译 `gzip_static` / 无 brotli | 配置直接报错 / 无预压 | **已确认** Ubuntu 26.04 有 `libnginx-mod-http-brotli-static`（可 apt 装）；`gzip_static` 须 `nginx -V` 校验 |
| **Ubuntu 26.04 的 nginx 包不支持 HTTP/3**（已实测确认，台账 4.13） | HTTP/3 不可用 | **接受**。HTTP/1.1+HTTP/2 已足够；如确需 HTTP/3，改用 nginx.org 官方二进制包（含 QUIC）或自编译——**列为二期可选项，不阻塞上线** |
| 我无法登录服务器 | 无法代执行 | 所有服务器操作产出为**可粘贴脚本**，你执行后回贴输出 |
