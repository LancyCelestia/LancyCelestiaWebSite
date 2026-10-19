# 事实核查台账

> 目的：把「查到的」与「推断的」分开，避免方案建立在编造之上。
> 置信度：**high** = 官方原文/registry 实测；**medium** = 官方但版本时间存疑，或权威二手；**low** = 无权威来源，**不采用为依据**。

---

## 1. ICP 备案（阿里云官方文档）

| # | 事实 | 置信度 | 来源 |
|---|---|---|---|
| 1.1 | 个人网站名称**全国禁用词**：「个人空间、爱好者、博客、导航、工作室、论坛、平台、热线、社区、社团、网络、网站、网址、主页、资讯、作品展示」；另「博客、论坛、在线、社区、交流」属禁止类别 | **high** | [阿里云·填写网站信息](https://help.aliyun.com/zh/icp-filing/basic-icp-service/user-guide/fill-in-website-information) |
| 1.2 | 名称基本要求：**≥3 个汉字**；禁纯数字/纯英文/域名；禁特殊符号；**不得使用人物名、地名或成语** | **high** | 同 1.1 |
| 1.3 | **浙江省额外禁用**：「彩票、产业、电商、工坊、股票、刮刮乐、官网、农业、企业、体彩、行业、养殖、在线、证券、中心」 | **high** | 同 1.1 |
| 1.4 | 「**茶馆**」「**终端**」「分享」**不在**禁用清单内（未受限 ≠ 已获准） | medium | 同 1.1（反向推断） |
| 1.5 | **备案不要求网站已可访问**。官方原文：「备案前您需准备合适的服务器，**备案成功后搭建网站**，为域名添加解析之后可通过域名访问网站」 | **high** | [备案流程 FAQ](https://help.aliyun.com/zh/icp-filing/basic-icp-service/support/for-the-record-process-faq) |
| 1.6 | 备案要求：域名解析指向**中国内地服务器**，须为阿里云中国内地节点且有购买时长要求 | **high** | 同 1.5 |

**影响**：
- 「在线」绝对不能出现在站名里（全国 + 浙江双禁）。
- 「澜汐个人终端」→ 4 个汉字 ✓、无禁用词 ✓，**但**「个人终端」是自造词，人工审核可能判为服务类；风险 medium。
- 「潮声茶馆」→ 4 汉字 ✓、无禁用词 ✓；「茶馆」有经营性联想，风险 medium。
- **网站开发与备案可并行**（1.5），不必串行等待。

---

## 2. 流量成本（决定省钱模型的第一性原则）

| # | 事实 | 置信度 | 来源 |
|---|---|---|---|
| 2.1 | **ECS 按流量出网 0.8 元/GB（华东1杭州），无免费额度**；入网不计费 | **high** | [公网带宽计费](https://help.aliyun.com/zh/ecs/public-bandwidth) |
| 2.2 | OSS 外网流出 **0.50 元/GB**（华东1杭州） | medium-high（官方计费案例，基准注 2024-11-20 单价） | [OSS 计费案例](https://help.aliyun.com/zh/oss/billing-examples) |
| 2.3 | OSS 新用户免费：标准存储 20GB / 外网流出 2GB / 请求 20 万次，**各 3 个月** | **high** | [免费额度](https://help.aliyun.com/zh/oss/free-quota-for-new-users) |
| 2.4 | OSS IMG 支持 WebP 与 AVIF：`?x-oss-process=image/format,webp\|avif`；**华东1杭州支持 AVIF**；原图宽高须 ≤16383px | **high** | [格式转换](https://help.aliyun.com/zh/oss/user-guide/convert-image-formats-2) |
| 2.5 | 阿里云 CDN 加速区域为「仅中国内地」或「全球」时，加速域名**必须备案** | **high** | [CDN 添加域名](https://help.aliyun.com/zh/cdn/add-a-domain-name) |
| 2.6 | DCDN/ESA 加速区域为「全球」或「仅中国内地」时，**无论源站在哪，域名都必须备案** | **high** | [DCDN 添加域名](https://www.alibabacloud.com/help/zh/dynamic-route-for-cdn/latest/add-a-domain-name) |

**省钱第一性原则（我的推断，非官方）**：
```
ECS 出网 0.8 元/GB  ← 最贵，第一优先砍
OSS 直出 0.5 元/GB
CDN 下行 + 回源 ≈ 0.39 元/GB
```
→ 一切设计目标是：**把字节搬离 ECS**。

---

## 3. 图片与外链的可用性

| # | 事实 | 置信度 | 来源 |
|---|---|---|---|
| 3.1 | jsDelivr **无中国大陆官方节点**，官网仅承诺 "Near China" 网络 | medium | [jsdelivr.com/network](https://www.jsdelivr.com/network) |
| 3.2 | jsDelivr 大陆可达性**历史性间歇中断**：2025-03~06、2025-11~2026-01 均有受干扰期 | **high** | [GreatFire](https://zh.greatfire.org/https/cdn.jsdelivr.net) |
| 3.3 | **ImgBB 大陆可访问性：无权威来源**，GreatFire 无记录 | **low｜不可断言** | — |
| 3.4 | 「境外图床为已备案境内网站供资源是否违规」：**无工信部/管局直接条文** | **low｜不采用坊间说法** | — |

**结论（我的判断）**：jsDelivr 与 ImgBB **都不能作为生产图床的主链路**。3.2 已证明它会在关键时刻挂掉，3.3 连能不能访问都无法证实。

---

## 4. 技术栈事实（官方/registry 实测）

| # | 事实 | 置信度 | 来源 |
|---|---|---|---|
| 4.1 | **Astro 最新稳定版 = 7.4.1**，`engines.node >= 22.22.0` | **high** | registry.npmjs.org/astro/latest |
| 4.2 | Astro 默认把 UI 组件渲染为 HTML/CSS，**自动剥离客户端 JS**；需显式 `client:*` 才注水 | **high** | [Islands](https://docs.astro.build/en/concepts/islands/) |
| 4.3 | `astro:assets` 内置 `<Image>/<Picture>/getImage()`，构建期生成多尺寸多格式，图片服务基于 **sharp** | **high** | [Images](https://docs.astro.build/en/guides/images/) |
| 4.4 | 官方 i18n **内置**：`i18n:{locales,defaultLocale}`、`routing.prefixDefaultLocale`（默认 false）、helper 来自 `astro:i18n` | **high** | [i18n](https://docs.astro.build/en/guides/internationalization/) |
| 4.5 | `ClientRouter`（View Transitions）存在，来自 `astro:transitions` | **high** | [View Transitions](https://docs.astro.build/en/guides/view-transitions/) |
| 4.6 | 默认构建产物为**纯静态** `dist/`，要 SSR 才装 adapter | **high** | [Deploy](https://docs.astro.build/en/guides/deploy/) |
| 4.7 | Brotli 最高质量 = **11**（`BROTLI_MAX_QUALITY`） | **high** | [brotli encode](https://brotli.org/encode.html) |
| 4.8 | Ubuntu 26.04 (resolute) universe 含 `libnginx-mod-http-brotli-static 1.0.0~rc-7build3` 与 `-filter`，**无需编译** | **high** | packages.ubuntu.com 实测 |
| 4.9 | 真实包名带 `lib` 前缀（`nginx-mod-...` 会 apt 失败） | **high** | 同上 |
| 4.10 | `gzip_static` 与 `brotli_static` **可共存**，各自按 `Accept-Encoding` 选 `.br`/`.gz` | **high** | [gzip_static](https://nginx.org/en/docs/ngx_http_gzip_static_module.html) |
| 4.11 | `ngx_http_v3_module`（HTTP/3）**默认不编译**，需 `--with-http_v3_module`；官方标注 experimental | **high** | [http_v3_module](https://nginx.org/en/docs/http/ngx_http_v3_module.html) |
| 4.12 | `gzip_static` 同样**默认不编译**（需 `--with-http_gzip_static_module`） | **high** | [gzip_static](https://nginx.org/en/docs/ngx_http_gzip_static_module.html) |
| 4.13 | **【已实测确认】Ubuntu 26.04 的 nginx 包不含 HTTP/3**：`nginx-core` 1.28.3-2ubuntu1.11 的 OPTIONAL HTTP MODULES 仅有 HTTP/2，`nginx-extras` / `nginx-full` 同样无 QUIC | **high** | [Launchpad nginx 1.28.3-2ubuntu1.11](https://launchpad.net/ubuntu/+source/nginx/1.28.3-2ubuntu1.11) |
| 4.13b | nginx.org 官方 **Linux 二进制包**自 1.25.0 起**已包含** QUIC/HTTP3（源码编译仍需 `--with-http_v3_module`） | **high** | [nginx QUIC](https://nginx.org/en/docs/quic.html) |
| 4.13c | `brotli_static` 与 `gzip_static` **可同时开启**（各自按 Accept-Encoding 协商）；但**两者同时开启时的优先级，nginx 官方未定义**（ngx_brotli 是第三方模块，nginx.org 不收录） | high（可共存）/ **未找到（优先级）** | [gzip_static](https://nginx.org/en/docs/ngx_http_gzip_static_module.html) |
| 4.14 | 本机 Node **v26.7.0** / npm 11.19.0 / pnpm 12.14.0 / git 2.56.0 / OpenSSH 10.5p1 | 实测 | — |
| 4.15 | **本机无 rsync**（Windows 原生无） | 实测 | — |
| 4.16 | `scp` **无增量能力**：OpenSSH 9.0 起默认走 SFTP 协议，仅 `-r` 递归；`sftp -a`/`reput` 只是**单文件断点续传**，非增量；`sftp -r` 不跟随软链 | **high** | [scp(1)](https://man.openbsd.org/scp) / [sftp(1)](https://man.openbsd.org/sftp) |
| 4.17 | **rsync 要求两端都有 rsync 二进制** → 只给服务器装 rsync 无用，本机没有链就断 | **high** | rsync 基本原理（同上调研） |
| 4.18 | Ubuntu 26.04 `rsync 3.4.1+ds1-7` 在 **main** 组件，`apt install rsync` 可用；是否预装未获官方明示 | high（可装）/ medium（预装） | [Launchpad](https://launchpad.net/ubuntu/+source/rsync/) |
| 4.19 | `ssh2` npm **1.17.0（2025-08-20）仍在维护**；`scp2` 0.5.0（2016-05-29）**已弃用**，依赖 ssh2 ~0.4.10，不可用 | **high** | registry.npmjs.org/ssh2、/scp2 |
| 4.20 | 阿里云 Workbench 有浏览器文件管理，单文件 ≤1200 MB，HTTPS **不支持断点续传**；官方建议大文件用 rsync/SFTP/OSS | **high** | [管理文件](https://help.aliyun.com/zh/ecs/user-guide/manage-files) |
| 4.21 | 产物提交 Git **应避免进主分支**，用 orphan 分支或独立仓库；且国内 ECS 直连 GitHub 常不稳 | medium | 调研结论 |

---

## 5. 压缩与字体（含「严禁编造」警戒线）

| # | 事实 | 置信度 | 来源 |
|---|---|---|---|
| 5.1 | RFC 7932 仅定性称 Brotli 比 gzip「considerably better」，**未给数字** | **high** | [RFC 7932](https://www.rfc-editor.org/rfc/rfc7932.html) |
| 5.2 | 业界常引「Brotli 比 gzip 小 15–25%」**无一手权威出处** → **禁止当作事实引用，须实测** | **low｜禁用** | — |
| 5.3 | **【实测更正】中文全量单字重 WOFF2 约 1.09 MiB（1,141,536 字节）**，不是 4–10 MB。来源：@fontsource/noto-sans-sc@5.2.8 的 `chinese-simplified-400-normal.woff2` 实测 Content-Length | **high（实测）** | cdn.jsdelivr.net/npm/@fontsource/noto-sans-sc@5.2.8 |
| 5.3b | Google Fonts CSS2 的 Noto Sans SC 400 **实测 101 个 `@font-face` 分片**；单片 13–41 KB；**一个中文页通常只命中 3–6 片，约 30–150 KB**（分片数实测为 high，「3–6 片」为按页面用字推断，medium） | high / medium | fonts.googleapis.com/css2?family=Noto+Sans+SC |
| 5.4 | web.dev 明确：亚洲字形多，完整字体以 MB 计；unicode-range 子集化对亚洲语言尤为重要 | **high** | [reduce-webfont-size](https://web.dev/articles/reduce-webfont-size) |
| 5.5 | **`font-display: swap` 会导致严重布局偏移、影响 CLS**；应优先 `optional`/`fallback` | **high** | 同 5.4 |
| 5.6 | `size-adjust` / `ascent-override` 用于消除回退字体的 CLS，Baseline 广泛可用（2023-09 起） | **high** | [MDN size-adjust](https://developer.mozilla.org/en-US/docs/Web/CSS/Reference/At-rules/@font-face/size-adjust) |
| 5.7 | 「系统字体栈是低成本合理做法」**仅找到 1 个权威来源**（web.dev），第二个独立权威源未找到 | medium | 同 5.4 |

---

## 6. HTTPS 证书（2026 年新变化，影响运维）

| # | 事实 | 置信度 | 来源 |
|---|---|---|---|
| 6.1 | 阿里云 SSL 证书自 **2026 年 2 月**起改为**订阅服务**模式；按 CA/B 新规，**单张证书有效期缩短至 200 天** | **high** | [阿里云公告（更新于 2026-10-09）](https://www.alibabacloud.com/help/zh/ssl-certificate/product-overview/notice-ssl-certificate-is-upgraded-to-subscription-service-mode) |
| 6.2 | 200 天有效期意味着**一年需续期约 2 次**，手动运维易漏 | 推断 | 由 6.1 推导 |
| 6.3 | 免费 DV 证书当前是否仍提供、额度多少：**未从官方页面核实到**（帮助文档 404） | **low｜待核实** | — |

**结论**：HTTPS 方案**不能依赖手动续期**。优先 **Let's Encrypt + certbot 自动续期**（90 天、自动 reload），把 200 天问题彻底绕开。

---

## 7. 缺口清单（写进方案的「待补充」）

- (待补充：补充项 1) ECS 的**备案服务号额度**现状（用户未使用过该功能）
- (待补充：补充项 2) 阿里云免费 DV 证书当前政策与额度
- (待补充：补充项 3) CDN / ESA 现行单价与是否有流量包（未核实到一手定价页）
- (待补充：补充项 4) Ubuntu 26.04 nginx 包的实际编译参数（须上机 `nginx -V`）
- (待补充：补充项 5) SSH 公钥或私钥路径；★ 注意：**关闭了审批**，我无法代你登录服务器，所有服务器操作只能由你在本地执行我给的脚本
