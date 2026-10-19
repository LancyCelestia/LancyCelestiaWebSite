### Task 16: 服务器基线（含内存护栏与模块探测）

**Files:** Create `deploy/scripts/00-baseline.sh`

**约束依据**：2 GiB + Artalk(~80MB) + submitd(~25MB) + Nginx(~30MB) ≈ 135MB，**必须主动防 OOM**。

- [ ] **Step 1: 写脚本**

```bash
#!/usr/bin/env bash
set -euo pipefail

apt-get update
apt-get install -y nginx certbot python3-certbot-nginx \
  libnginx-mod-http-brotli-static libnginx-mod-http-brotli-filter

# 2 GB swap：2 GiB 机器的 OOM 保险丝
if ! swapon --show | grep -q '/swapfile'; then
  fallocate -l 2G /swapfile && chmod 600 /swapfile && mkswap /swapfile && swapon /swapfile
  grep -q '/swapfile' /etc/fstab || echo '/swapfile none swap sw 0 0' >> /etc/fstab
fi
sysctl -w vm.swappiness=10
grep -q 'vm.swappiness' /etc/sysctl.conf || echo 'vm.swappiness=10' >> /etc/sysctl.conf

# 服务内存硬限（只用 MemoryMax，不用 MemoryHigh：软限会触发 Go 运行时频繁 GC）
mkdir -p /etc/systemd/system/artalk.service.d /etc/systemd/system/submitd.service.d
cat > /etc/systemd/system/artalk.service.d/limits.conf <<'EOF'
[Service]
MemoryMax=256M
Restart=on-failure
EOF
cat > /etc/systemd/system/submitd.service.d/limits.conf <<'EOF'
[Service]
MemoryMax=96M
Restart=on-failure
EOF

# 部署目录与数据目录
mkdir -p /var/www/lancycelestia/releases /var/lib/submitd /var/lib/artalk
chown -R www-data:www-data /var/www/lancycelestia
chown -R artalk:artalk /var/lib/artalk 2>/dev/null || true
chown -R www-data:www-data /var/lib/submitd
```

- [ ] **Step 2: 用户执行并回贴三项输出**

```bash
nginx -V 2>&1 | tr ' ' '\n' | grep -E 'brotli|gzip_static|http_v3' || echo 'NONE'
free -h && swapon --show
```

- [ ] **Step 3: 依据输出决定 Nginx 模块指令**

| 输出 | 处理 |
|---|---|
| 含 `brotli` | Task 17 启用 `brotli_static on;` |
| **无** `gzip_static` | Task 16 **不得**写 `gzip_static on;`（否则 `nginx -t` 报 unknown directive） |
| **无** `http_v3` | 预期内（台账 4.13），配置中**不得**出现 `quic`/`http3` |
| `swapon` 为空 | Step 1 失败，**回到 Step 1 重跑**，不得继续 |

---

