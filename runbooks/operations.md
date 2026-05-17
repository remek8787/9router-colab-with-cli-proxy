# Operations Runbook — ACSA 9Router Managed Proxy

## Service paths

```text
Managed proxy app: /home/ubuntu/managed-proxy
Managed proxy service: acsa-managed-proxy.service
Managed proxy DB: /home/ubuntu/managed-proxy/proxy.db
Nginx site: /etc/nginx/sites-available/acsa.my.id
9Router data: /home/ubuntu/.9router
```

## Health checks

```bash
systemctl is-active acsa-managed-proxy
systemctl is-active 9router
sudo nginx -t
curl -I https://acsa.my.id/cekusage
curl -s -o /dev/null -w '%{http_code}\n' https://acsa.my.id/v1/models
```

Expected:

- `acsa-managed-proxy` active
- `9router` active
- `nginx -t` successful
- `/cekusage` returns `200`
- `/v1/models` without token returns `401`

## Safe deploy checklist

1. Backup file before editing:

```bash
cd /home/ubuntu/managed-proxy
cp server.mjs server.mjs.bak-$(date +%Y%m%d-%H%M%S)
sudo cp /etc/nginx/sites-available/acsa.my.id /etc/nginx/sites-available/acsa.my.id.bak-$(date +%Y%m%d-%H%M%S)
```

2. Syntax checks:

```bash
node --check /home/ubuntu/managed-proxy/server.mjs
sudo nginx -t
```

3. Restart/reload:

```bash
sudo systemctl restart acsa-managed-proxy
sudo systemctl reload nginx
```

4. Smoke test:

```bash
curl -I https://acsa.my.id/cekusage
curl -s -o /dev/null -w '%{http_code}\n' https://acsa.my.id/v1/models
```

## `/cekusage` behavior

- Public no-login page.
- User provides token in POST body.
- Backend hashes token and returns only matching key usage.
- Supported periods: today, 7 days, current month.
- Invalid token returns a friendly not-found page, not a server error.

## Do not commit

- `/home/ubuntu/managed-proxy/admin.env`
- `/home/ubuntu/managed-proxy/proxy.db*`
- `/home/ubuntu/.9router/db/*`
- raw tokens/passwords/API keys
