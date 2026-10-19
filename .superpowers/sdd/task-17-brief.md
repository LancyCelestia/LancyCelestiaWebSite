### Task 17: Nginx 配置（www/裸域发副本，不做 301）

**Files:** Create `deploy/nginx/blog.conf`

**决策依据**：用户要求 `www` 与裸域**各放一份 blog 页面副本，不做 301**。用 `canonical` 解决重复内容。

- [ ] **Step 1: 写配置（模块指令按 Task 16 输出条件启用）**

```nginx
# ---------- 通用压缩设置 ----------
# 若 Task 16 输出无 gzip_static，删除 gzip_static 行
brotli_static on;
gzip_static on;
brotli_types text/html text/css application/javascript application/json image/svg+xml application/xml;
gzip_types text/html text/css application/javascript application/json image/svg+xml application/xml;

limit_req_zone $binary_remote_addr zone=perip:10m rate=10r/s;

# ---------- blog.lancycelestia.com（主站） ----------
server {
    listen 80;
    server_name blog.lancycelestia.com;
    root /var/www/lancycelestia/current;
    index index.html;

    # Brotli/Gzip 协商
    add_header Vary Accept-Encoding;

    # 带 hash 的静态资源：一年 immutable
    location /_assets/ {
        add_header Cache-Control "public, max-age=31536000, immutable" always;
        try_files $uri =404;
    }

    # HTML：短缓存
    location ~* \.html$ {
        add_header Cache-Control "public, max-age=300, must-revalidate" always;
        try_files $uri =404;
    }

    # 评论：Artalk 反代（注意结尾不带 /，保留 /api/v2 原样）
    location /api/v2/ {
        proxy_pass http://127.0.0.1:23366;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # 投稿：自研 submitd
    location /api/submit/ {
        limit_req zone=perip burst=5 nodelay;
        rewrite ^/api/submit/(.*)$ /$1 break;
        proxy_pass http://127.0.0.1:23367;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        client_max_body_size 64k;
    }

    # 搜索索引（Pagefind）
    location /pagefind/ {
        add_header Cache-Control "public, max-age=3600" always;
        try_files $uri =404;
    }

    location / {
        try_files $uri $uri/ $uri.html =404;
    }

    # 防爬
    location ~ /\. { deny all; }
    if ($http_user_agent ~* (semrush|ahrefs|mj12bot|dotbot)) { return 403; }
}

# ---------- www 与裸域：同一份内容副本，不做跳转 ----------
server {
    listen 80;
    server_name www.lancycelestia.com lancycelestia.com;
    root /var/www/lancycelestia/current;   # 与主站同一份产物
    index index.html;
    add_header Vary Accept-Encoding;
    location /_assets/ { add_header Cache-Control "public, max-age=31536000, immutable" always; try_files $uri =404; }
    location / { try_files $uri $uri/ $uri.html =404; }
    # 注意：此处不得出现 return 301 / rewrite ... https://blog. ，用户明确要求不跳转
}
```

- [ ] **Step 2: 用户执行**

```bash
cp deploy/nginx/blog.conf /etc/nginx/sites-available/blog.conf
ln -sf /etc/nginx/sites-available/blog.conf /etc/nginx/sites-enabled/blog.conf
rm -f /etc/nginx/sites-enabled/default
nginx -t
systemctl reload nginx
```

- [ ] **Step 3: 验证（三条硬性）**

```bash
# 1) brotli 生效
curl -sI -H 'Accept-Encoding: br' https://blog.lancycelestia.com/index.html | grep -i content-encoding
# 2) www 与裸域不得 301 到 blog.
curl -sI http://www.lancycelestia.com | grep -iE '^(HTTP|location)'
curl -sI http://lancycelestia.com | grep -iE '^(HTTP|location)'
# 期望：HTTP/1.1 200，且无 location: https://blog.…
```

---

