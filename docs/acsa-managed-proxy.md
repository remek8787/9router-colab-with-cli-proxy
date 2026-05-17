---
title: "Blueprint — acsa.my.id Managed Proxy for 9Router"
description: "Per-client API key, request/token limit, and usage tracking layer in front of 9Router on acsa.my.id. Secrets are not stored here."
project: openclaw, acsa, 9router, managed-proxy
created: "2026-05-17"
updated: "2026-05-17"
tags: [acsa.my.id, 9router, api-key, quota, proxy, usage]
---

# Blueprint — acsa.my.id Managed Proxy for 9Router

## Goal

Expose an OpenAI-compatible endpoint on `https://acsa.my.id/v1/*` where each client/machine uses its own API token. The proxy validates the token, enforces per-token limits, records usage, and forwards allowed requests to local 9Router.

## Runtime on acsa.my.id

- Public API endpoint: `https://acsa.my.id/v1/*`
- Public dashboard/root: `https://acsa.my.id/` still points to 9Router
- Managed proxy service: `acsa-managed-proxy.service`
- Managed proxy listen: `127.0.0.1:8790`
- Upstream 9Router: `127.0.0.1:20128/v1`
- Proxy app dir: `/home/ubuntu/managed-proxy`
- Proxy DB: `/home/ubuntu/managed-proxy/proxy.db`
- CLI: `/usr/local/bin/acsa-proxyctl`
- Admin token file on server only: `/home/ubuntu/managed-proxy/admin.env`

## Supported features

- API key per client/machine.
- Per-key active/disabled flag.
- Per-key allowed models:
  - `*` for all models exposed by 9Router.
  - Comma-separated list for restricted models, e.g. `acspt/gpt-5.5,acspt/gpt-5.4-mini`.
- Per-key request-per-minute limit (`rpm`).
- Per-key request-per-day limit (`rpd`).
- Per-key token-per-day limit (`tpd`, `0` = disabled).
- Usage summary by key prefix and model.
- Recent usage events.

## CLI examples

Create a token:

```bash
acsa-proxyctl create client-a 60 10000 0 '*'
```

Create a restricted token:

```bash
acsa-proxyctl create client-b 30 1000 500000 'acspt/gpt-5.5'
```

List keys:

```bash
acsa-proxyctl keys
```

Update limits:

```bash
acsa-proxyctl update <id> rpm=30 rpd=1000 tpd=500000 allowed_models=acspt/gpt-5.5 active=true
```

Disable a key:

```bash
acsa-proxyctl update <id> active=false
```

Usage summary:

```bash
acsa-proxyctl usage
```

Recent events:

```bash
acsa-proxyctl events 50
```

## Client usage

```bash
curl https://acsa.my.id/v1/chat/completions \
  -H "Authorization: Bearer <client-token>" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "acspt/gpt-5.5",
    "messages": [{"role":"user","content":"Halo"}],
    "stream": false
  }'
```

## Verification result on 2026-05-17

- No token: `401`.
- Valid token: `/v1/models` returned models from 9Router.
- Valid token chat: `acspt/gpt-5.5` returned `OK`.
- RPM test: token with `rpm=2` returned `429` on third quick request.
- Test tokens were disabled after verification.

## Provider/model management

Current design keeps 9Router as the model/provider control plane. Adding provider/model in 9Router should automatically appear behind `https://acsa.my.id/v1/models`, while the managed proxy controls access and limits.

Future improvement: add a small admin web UI for `acsa-proxyctl` functions.

## Web UI added on 2026-05-17

Admin UI:

```text
https://acsa.my.id/proxy/
```

Login credentials were set on the remote server via systemd env only, not stored in this note. UI supports:

- Login/logout.
- Create generated API token.
- Import existing 9Router-style token such as `sk-...` by pasting it into the token field.
- Set `rpm`, `rpd`, `tpd`, and `allowed_models` per token.
- Enable/disable token.
- Reset usage per token.
- View today usage per token/model including success and failed request counts.

Smoke test result: login succeeded, dashboard loaded, import token route worked, update limit route worked, reset usage route worked, and public `/v1/models` without token returns `401`.

## 9Router key auto-sync added on 2026-05-17

Managed proxy now reads `/home/ubuntu/.9router/db/data.sqlite` and syncs 9Router API keys into its own DB by hash. This means keys created in 9Router on the same server can appear in the managed proxy dashboard with their 9Router names.

Sync behavior:

- Runs once at managed proxy startup.
- Runs every 60 seconds.
- Also runs when opening the authenticated `/proxy/` dashboard.
- Manual button: `Sync 9Router Keys`.

Dashboard now shows:

- Token name from 9Router, e.g. `OM-YUYUS`, `OPC-AlVI`, `OPCKU`.
- Source badge: `9router`, `manual`, or `imported`.
- Machine ID for 9Router keys.
- Editable per-token limits and allowed model list.

Cleanup: temporary smoke-test keys were removed from the managed proxy DB so the dashboard stays readable.

## UI redesign on 2026-05-17

Tuan Besar provided a visual reference for a cleaner proxy dashboard. The managed proxy `/proxy/` UI was redesigned to use:

- Dark fixed sidebar with app branding and navigation.
- Light main dashboard canvas.
- Summary metric cards: total token, today's requests, failed requests, total AI tokens.
- Cleaner create/import token card.
- Status proxy card.
- Token & Limits table with source/name/prefix/status/limit/model/usage/actions.
- Usage by Token / Model table.
- More compact action buttons and safer horizontal table scrolling so right-side buttons do not get clipped.

Smoke test passed after redesign:

- Login works.
- Dashboard contains expected markers: Proxy Management, Dashboard Proxy, Tokens & Limits, Usage by Token / Model.
- Auto-synced 9Router key names visible: OM-YUYUS, OPC-AlVI, OPCKU.
- `/v1/models` without token remains 401.
- Services active: acsa-managed-proxy, 9router, nginx.

Git reference requested by Tuan Besar: `git@github.com:remek8787/9router-colab-with-cli-proxy.git`. The repo was cloned locally under `/root/.openclaw/workspace/projects/9router-colab-with-cli-proxy`, but it was empty at clone time (no commits yet).

## Dashboard status/time update on 2026-05-17

Tuan Besar requested clearer dashboard indicators and Asia/Jakarta time.

Changes:

- Managed proxy process uses `TZ=Asia/Jakarta`.
- Dashboard copy states usage times are shown in WIB / Asia/Jakarta.
- Usage timestamps are formatted with `Asia/Jakarta` and appended with `WIB`.
- 9Router synced keys that disappear from 9Router are no longer hard-deleted from managed proxy; they are marked inactive with source badge `removed`, preserving usage history.
- Dashboard status cards show 9Router/managed API/protected endpoint summary.

Observed state after sync:

- Active 9Router keys: `OM-YUYUS`, `OPC-AlVI`.
- Removed/kept for history: `OPCKU`.

## Removed-token delete behavior on 2026-05-17

Tuan Besar requested that tokens which no longer exist in 9Router and are marked `removed` may be deleted from the managed proxy dashboard. Implemented:

- Dashboard action column can show a `Delete` button for `removed`, `manual`, or `imported` tokens.
- Active synced `9router` tokens cannot be deleted from managed proxy directly; delete them in 9Router first, then sync marks them removed.
- Existing removed key `OPCKU` was deleted from managed proxy DB after request.
- Verification: dashboard no longer shows OPCKU, while active synced keys remain visible. Current keys included OM-HERI, OM-YUYUS, and OPC-AlVI.

## Public `/cekusage` token usage checker added on 2026-05-18

Tuan Besar requested a separate public page so usage checking does not interfere with `/proxy/` admin dashboard or `/v1/*` API routes.

Public page:

```text
https://acsa.my.id/cekusage
```

Behavior:

- No login is required for the page.
- User enters their own API token in a form.
- Token is submitted via `POST` body, not query string, so it does not appear in the browser address bar.
- Backend hashes the submitted token and only returns usage data for the matching token.
- If token is invalid, page returns a normal friendly “token not found” result.
- The page does **not** list all tokens and does **not** expose raw tokens.

Displayed usage sections:

- Token name/status.
- Request count.
- Success/fail count.
- Prompt tokens.
- Completion tokens.
- Total tokens.
- Last usage timestamp in WIB formatting.
- Breakdown by day.
- Breakdown by model.
- 10 latest requests for that token.

Supported period filter:

- `today` / Hari ini.
- `7d` / 7 hari terakhir.
- `month` / Bulan ini.

Implementation notes:

- App file patched on server: `/home/ubuntu/managed-proxy/server.mjs`.
- Backup made before deployment: `server.mjs.bak-cekusage-*`.
- Nginx route added in `/etc/nginx/sites-available/acsa.my.id`:
  - `location = /cekusage` → `http://127.0.0.1:8790/cekusage`.
- Nginx backup made before route patch: `acsa.my.id.bak-cekusage-*`.
- Managed proxy service restarted successfully after syntax check.

Smoke test result:

- `GET https://acsa.my.id/cekusage` → `200 OK`.
- `POST https://acsa.my.id/cekusage` with valid active 9Router token → `200 OK`, result contains token name, usage summary, and breakdown model.
- `POST https://acsa.my.id/cekusage` with invalid token → `200 OK`, result says token not found.
- `GET https://acsa.my.id/v1/models` without token remains `401`.
- `acsa-managed-proxy.service` remains active.
- `nginx -t` passes.

SSH access update:

- Public key from this OpenClaw host was added to `ubuntu@43.134.122.109` for future maintenance.
- Password/token secrets must not be written into repos, docs, memory, or public logs.
