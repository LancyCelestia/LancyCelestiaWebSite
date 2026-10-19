### Task 20: HTTPS（备案通过后才做）

**前提**：Task 18 备案已通过，且 80 端口可被公网访问。

- [ ] **Step 1: 签发证书**

```bash
certbot --nginx -d blog.lancycelestia.com -d www.lancycelestia.com -d lancycelestia.com \
  --redirect --agree-tos -m <你的邮箱> --no-eff-email
```
> **注意**：`--redirect` 只会把**同域名**的 http 跳到 https，**不会**把 `www` 跳到 `blog`（因为它们是不同的 `server_name` 块）。签发后**必须验证**这一条。

- [ ] **Step 2: 验证约束 1（www/裸域不得 301 到 blog）**

```bash
for h in www.lancycelestia.com lancycelestia.com; do
  echo "== $h =="
  curl -sI http://$h | grep -iE '^(HTTP|location)'
  curl -sIL https://$h | grep -iE '^(HTTP|location)'
done
# 通过标准：Location 只能出现 https://<同一个 host>/…，不得出现 blog.lancycelestia.com
```

- [ ] **Step 3: 验证自动续期**

```bash
certbot renew --dry-run
systemctl list-timers | grep certbot
```

- [ ] **Step 4: SSL Labs 评级**

访问 `https://www.ssllabs.com/ssltest/analyze.html?d=blog.lancycelestia.com`，期望 **A 及以上**。

---

