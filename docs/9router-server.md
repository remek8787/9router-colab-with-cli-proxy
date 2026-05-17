---
title: "Blueprint — 9Router Server 10.3.16.55"
description: "Catatan instalasi 9Router pada server Ubuntu 10.3.16.55. Tidak menyimpan password/kredensial plaintext."
project: openclaw, infrastructure, 9router
created: "2026-05-17"
updated: "2026-05-17"
tags: [9router, ubuntu, ai-router, systemd, openai-compatible]
---

# Blueprint — 9Router Server 10.3.16.55

## Status

Installed and verified on 2026-05-17.

- Local/private server IP: `10.3.16.55`
- Public IP / NAT: `43.134.122.109`
- Public domain: `acsa.my.id`
- SSH port: `22`
- SSH user used for install: `ubuntu`
- OS: Ubuntu 24.04.4 LTS Noble
- Architecture: `x86_64`
- RAM observed: ~3.6 GiB
- Disk root observed: ~59 GiB total, ~52 GiB free before install
- 9Router version installed: `0.4.50`
- Node.js installed: `v22.22.2`
- npm installed: `10.9.7`
- Nginx installed as reverse proxy
- HTTPS installed via Let's Encrypt / Certbot
- Certificate expiry observed: 2026-08-15 00:09:32 UTC

Credential note: password was provided by Tuan Besar in chat for this session only. Do **not** write it into files, repos, blueprints, scripts, or public logs.

## Installed service

9Router is installed globally via npm:

```bash
sudo npm install -g 9router
```

Binary path:

```bash
/usr/bin/9router
```

Systemd service:

```text
/etc/systemd/system/9router.service
```

Current service design:

```ini
[Unit]
Description=9Router AI Router
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=ubuntu
WorkingDirectory=/home/ubuntu
Environment=PORT=20128
Environment=HOSTNAME=0.0.0.0
Environment=BASE_URL=http://10.3.16.55:20128
Environment=NEXT_PUBLIC_BASE_URL=http://10.3.16.55:20128
ExecStart=/usr/bin/9router --host 0.0.0.0 --port 20128 --no-browser --log --skip-update
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Important gotcha learned during install:

- Running plain `/usr/bin/9router` under systemd starts then exits after a few seconds.
- Stable systemd mode needs explicit flags:
  - `--host 0.0.0.0`
  - `--port 20128`
  - `--no-browser`
  - `--log`
  - `--skip-update`

## Runtime endpoints

Primary public dashboard:

```text
https://acsa.my.id/dashboard
```

Primary public login page:

```text
https://acsa.my.id/login
```

Primary public OpenAI-compatible API base:

```text
https://acsa.my.id/v1
```

Internal/direct dashboard:

```text
http://10.3.16.55:20128/dashboard
```

Internal/direct login page:

```text
http://10.3.16.55:20128/login
```

Internal/direct OpenAI-compatible API base:

```text
http://10.3.16.55:20128/v1
```

Observed verification:

```text
GET http://acsa.my.id/dashboard -> HTTP/1.1 301 Moved Permanently, Location: https://acsa.my.id/dashboard
GET https://acsa.my.id/dashboard -> HTTP/1.1 307 Temporary Redirect, location: /login
GET https://acsa.my.id/login -> HTML returned
Port listeners -> 0.0.0.0:80, 0.0.0.0:443, 0.0.0.0:20128
```

## Nginx / HTTPS proxy

Nginx site file:

```text
/etc/nginx/sites-available/acsa.my.id
/etc/nginx/sites-enabled/acsa.my.id
```

Proxy target:

```text
http://127.0.0.1:20128
```

Certbot certificate paths:

```text
/etc/letsencrypt/live/acsa.my.id/fullchain.pem
/etc/letsencrypt/live/acsa.my.id/privkey.pem
```

Certbot installed its renewal timer:

```bash
systemctl list-timers certbot.timer --no-pager
```

Nginx smoke tests:

```bash
curl -I http://acsa.my.id/dashboard
curl -I https://acsa.my.id/dashboard
curl -I https://acsa.my.id/login
```

## Data files

User data lives under:

```text
/home/ubuntu/.9router
```

Observed files after first launch:

```text
/home/ubuntu/.9router/db/data.sqlite
/home/ubuntu/.9router/db/data.sqlite-shm
/home/ubuntu/.9router/db/data.sqlite-wal
/home/ubuntu/.9router/jwt-secret
/home/ubuntu/.9router/runtime/package.json
```

Approx size after first launch: ~12 MB.

Backup target before risky changes:

```bash
tar -czf ~/9router-backup-$(date +%F-%H%M).tar.gz ~/.9router /etc/systemd/system/9router.service
```

## Useful commands

Check service:

```bash
systemctl status 9router --no-pager
systemctl is-active 9router
systemctl is-enabled 9router
```

Restart:

```bash
sudo systemctl restart 9router
```

Stop/start:

```bash
sudo systemctl stop 9router
sudo systemctl start 9router
```

Logs:

```bash
journalctl -u 9router -n 100 --no-pager
journalctl -u 9router -f
```

Check ports:

```bash
ss -lntp | grep -E ':(80|443|20128)'
```

HTTP smoke test from server:

```bash
curl -I http://127.0.0.1:20128/dashboard
curl -I http://10.3.16.55:20128/dashboard
```

HTTP smoke test from another machine with route to server:

```bash
curl -I http://10.3.16.55:20128/dashboard
```

## Setup notes

Node.js was installed from NodeSource Node 22 repo:

```bash
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://deb.nodesource.com/gpgkey/nodesource-repo.gpg.key -o /tmp/nodesource.gpg.key
sudo gpg --dearmor --yes -o /etc/apt/keyrings/nodesource.gpg /tmp/nodesource.gpg.key
printf '%s\n' 'deb [signed-by=/etc/apt/keyrings/nodesource.gpg] https://deb.nodesource.com/node_22.x nodistro main' | sudo tee /etc/apt/sources.list.d/nodesource.list >/dev/null
sudo apt-get update
sudo apt-get install -y nodejs
```

Install issue encountered:

- First NodeSource attempt produced `gpg: no valid OpenPGP data found` because a sudo/password pipeline was malformed.
- Second attempt fixed it by downloading key to `/tmp/nodesource.gpg.key` first, then dearmoring with sudo.

## Security notes / recommended next steps

Current install binds `0.0.0.0:20128`, and public HTTPS is exposed through Nginx on `https://acsa.my.id`.

Recommended hardening:

1. Prefer using public HTTPS endpoint `https://acsa.my.id` instead of direct port `20128`.
2. If possible, restrict direct public access to port `20128` at firewall/security-group/NAT level and leave only `80/443` public.
3. Do not publish API keys or provider tokens in repo/files.
4. Backup `/home/ubuntu/.9router` before upgrades or provider/account changes.
5. Consider UFW allowlist if server firewall is enabled:

```bash
sudo ufw allow from <trusted-ip-or-subnet> to any port 20128 proto tcp
```

Do **not** blindly enable broad public access unless Tuan Besar explicitly approves.

## Relationship to existing 9Router reference

General concept/reference blueprint remains:

```text
/root/.openclaw/workspace/notes/9router-reference-blueprint.md
```

This file is the server-specific deployment blueprint for `10.3.16.55`.
