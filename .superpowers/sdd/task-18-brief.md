### Task 18: 部署脚本（tar + scp + 原子软链切换）

**Files:** Create `scripts/deploy.sh`

- [ ] **Step 1: 写脚本**

```bash
#!/usr/bin/env bash
set -euo pipefail
HOST="${DEPLOY_HOST:?set DEPLOY_HOST}"
USER_="${DEPLOY_USER:-deploy}"
TS=$(date +%Y%m%d%H%M%S)
REMOTE=/var/www/lancycelestia

pnpm build && pnpm compress
tar -czf /tmp/dist-$TS.tar.gz -C dist .

scp /tmp/dist-$TS.tar.gz "$USER_@$HOST:/tmp/"

ssh "$USER_@$HOST" bash -s <<EOF
set -euo pipefail
mkdir -p $REMOTE/releases/$TS
tar -xzf /tmp/dist-$TS.tar.gz -C $REMOTE/releases/$TS
chown -R www-data:www-data $REMOTE/releases/$TS
ln -sfn $REMOTE/releases/$TS $REMOTE/current
ls -1dt $REMOTE/releases/* | tail -n +6 | xargs -r rm -rf
rm -f /tmp/dist-$TS.tar.gz
EOF
rm -f /tmp/dist-$TS.tar.gz
echo "deployed $TS"
```

- [ ] **Step 2: 执行并验证零停机**

```bash
DEPLOY_HOST=<服务器IP> bash scripts/deploy.sh
# 另开一个终端持续请求，确认切换瞬间无 404
while true; do curl -s -o /dev/null -w "%{http_code}\n" https://blog.lancycelestia.com/; sleep 0.2; done
```

- [ ] **Step 3: 回滚演练（必须有命令）**

```bash
ssh user@host 'ls -1dt /var/www/lancycelestia/releases/*'
ssh user@host 'ln -sfn /var/www/lancycelestia/releases/<上一版时间戳> /var/www/lancycelestia/current'
```

---

