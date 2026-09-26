---
title: "Blueprint — acsa.my.id Managed Proxy for 9Router"
description: "Per-client API key, request/token limit, and usage tracking layer in front of 9Router on acsa.my.id. Secrets are not stored here."
project: openclaw, acsa, 9router, managed-proxy
created: "2026-05-17"
updated: "2026-09-17"
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

## Success-only window quota + multiplier added on 2026-07-12

Tuan Besar requested per-token request quota like `30 request / 5 jam`, where failed requests do **not** reduce quota, plus quota multiplier options `x1` to `x5`.

Implemented on live server `/home/ubuntu/managed-proxy/server.mjs`:

- `api_keys.quota_multiplier INTEGER DEFAULT 1` added.
- Window quota uses:
  - `rpw` = base request quota.
  - `window_hours` = window duration.
  - `quota_multiplier` = `x1` to `x5`.
  - Effective quota = `rpw * quota_multiplier`.
- Example:
  - `rpw=30`, `window_hours=5`, `quota_multiplier=1` → 30 successful requests / 5 hours.
  - `rpw=30`, `window_hours=5`, `quota_multiplier=5` → 150 successful requests / 5 hours.
- Limit checks for RPM, RPD, and window quota now count only successful upstream responses (`status 200–299`).
- Failed/disallowed/upstream-error/429 events remain recorded in `usage_events`, but they do **not** reduce request quota.
- Dashboard token form now exposes base quota, window hours, and multiplier select `x1–x5`.
- Dashboard guide text was updated to explain success-only quota and multiplier.
- Admin JSON endpoints include `quota_multiplier` in create/list/update flows.

Deployment safety:

- Backup before patch on server:
  - `/home/ubuntu/managed-proxy/server.mjs.bak-success-quota-20260712-144922`
  - `/home/ubuntu/managed-proxy/proxy.db.bak-success-quota-20260712-144922`
- Syntax check passed before replacing live file.
- Service restarted successfully: `acsa-managed-proxy.service` active.

Smoke test result:

- Temporary token with `rpw=1`, `window_hours=5`, `quota_multiplier=2`.
- First request intentionally failed with model restriction: `403` and did not consume quota.
- Next two `/v1/models` requests succeeded: `200`, `200`.
- Third successful attempt blocked: `429 rate_limit_window`.
- DB events during test: `403`, `200`, `200`, `429`; success count was `2`, total events `4`.
- Temporary smoke-test token and events were deleted after verification.

## Non-destructive quota reset added on 2026-07-12

Tuan Besar requested the dashboard `Reset` action to return a token quota back to zero without deleting usage records. Implemented as a quota checkpoint instead of destructive log deletion.

Implementation:

- Added `api_keys.quota_reset_at TEXT`.
- Added `api_keys.quota_reset_after_id INTEGER DEFAULT 0`.
- Reset button now updates:
  - `quota_reset_at = CURRENT_TIMESTAMP`
  - `quota_reset_after_id = max(usage_events.id)` for that token.
- Window quota counting now filters by both:
  - current time window / manual reset timestamp.
  - `usage_events.id > quota_reset_after_id`.
- Existing usage records remain in `usage_events` for history/reporting.
- Dashboard shows reset marker: `Reset manual: ... after #...`.
- `/admin/reset` now also uses checkpoint reset and returns `records_preserved: true`.

Why `quota_reset_after_id` exists:

- SQLite timestamps are only second-level in this app.
- If reset happens in the same second as a request, timestamp-only reset can still count the old request.
- `quota_reset_after_id` makes reset exact: all records up to the last event ID are preserved but ignored for the next quota count.

Deployment safety:

- First checkpoint patch backup:
  - `/home/ubuntu/managed-proxy/server.mjs.bak-reset-checkpoint-20260712-145540`
  - `/home/ubuntu/managed-proxy/proxy.db.bak-reset-checkpoint-20260712-145540`
- Final robust after-ID patch backup:
  - `/home/ubuntu/managed-proxy/server.mjs.bak-reset-after-id-20260712-145929`
  - `/home/ubuntu/managed-proxy/proxy.db.bak-reset-after-id-20260712-145929`
- Service restarted successfully: `acsa-managed-proxy.service` active.

Smoke test result:

- Temporary token with quota `1 / 5 jam x1`.
- Before reset: `200`, then `429`.
- Reset checkpoint stored `quota_reset_after_id` at the last old event.
- After reset: next request returned `200`, then another request returned `429`.
- Records remained preserved: old and new events still existed during the test, with old events ignored by quota count after reset.
- Temporary smoke-test token and events were deleted after verification.

## 9Router model ID and usage synchronization — 2026-09-26

Canonical main model IDs from the 9Router `GC-Provider` catalog:

- `gcpr/gpt-5.6-luna` → 1x request
- `gcpr/gpt-5.6-terra` → 6x request
- `gcpr/gpt-5.6-sol` → 4x request
- `gcpr/gpt-6-astra` → 10x request
- `gcpr/gpt-5.5` → 1x request

`/cekusage` now displays the public endpoint, exact model ID, model cost, and source. It reads the canonical table plus 9Router combo records, without exposing credentials. Auto-refresh is 3 minutes / 180 seconds. Failed/non-2xx requests and `/v1/models` remain zero-cost; historical usage is preserved.

Live backup: `/home/ubuntu/backups/acsa-model-sync-20260926-20260926-125359/`. Correction backup for GPT-5.5: `/home/ubuntu/backups/acsa-model-weight-gpt55-20260926-130752/`. Coret map updated under the synchronization branch. The live `gcpr/gpt-5.5` mapping is now exactly 1x request.

## Request-weighted model cost update — 2026-09-26

Tuan Besar menetapkan bobot request baru yang diterapkan pada source live `/home/ubuntu/managed-proxy/server.mjs`, sehingga `/proxy/` dan `/cekusage` membaca aturan yang sama:

- GPT-6 Astra: `10x` request.
- GPT-5.6 Terra: `6x` request.
- GPT-5.6 Sol: `4x` request.
- GPT-5.6 Luna: `1x` request.

Request gagal/non-2xx dan `/v1/models` tidak mengurangi kuota. Histori lama dipertahankan; bobot baru berlaku untuk request sukses setelah deployment.

Backup live: `/home/ubuntu/backups/acsa-model-weight-20260926-20260926-124046/`.

Verifikasi: source lolos `node --check`, service active/running, `/proxy/` dan `/cekusage` HTTP 200, `/v1/models` tanpa token HTTP 401.

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

## Weekly Sol-eq quota system — 2026-07-13

The previous success-only hourly window quota was replaced by a provider-aligned weekly Sol-eq quota.

Current rules:

- Global pool: `3000 Sol-eq / 7 days × provider multiplier x1–x5`.
- Per-key quota: `weekly_sol_base × quota_multiplier x1–x5`.
- Optional daily Sol-eq guard prevents one key from exhausting its week in one day.
- RPM remains only as technical burst protection.
- Request failures and `/v1/models` cost zero Sol-eq.
- GPT-5.4 weight: 0.75; standard billable models: 1.0.
- Limit checks use projected request weight, so a request is rejected before it can exceed quota.
- The global pool protects the upstream even when individual key allocations overlap.

Initial active-key migration:

- Six active keys.
- Each: 500 Sol-eq/week, 150 Sol-eq/day, multiplier x1, RPM 60.
- Legacy RPD/TPD/RPW window controls disabled.

UI:

- `/proxy/` manages provider pool and per-key weekly/daily Sol-eq limits.
- `/cekusage` shows weekly remaining quota, daily guard, global pool, reset time, and Sol-eq breakdown.

Backup before migration:

- `/home/ubuntu/backups/acsa-managed-proxy-before-weekly-sol-20260713-203442`
- `/home/ubuntu/backups/acsa-managed-proxy-db-before-weekly-sol-20260713-203442.sqlite`

Canonical operational and rollback detail:
`/root/.openclaw/workspace/notes/acsa-managed-proxy-blueprint.md`


## Model-weighted usage update — 2026-09-17

Tuan Besar supplied a revised provider cost reference and requested both `/proxy/` and `/cekusage` to follow it.

Active successful-request weights:

- GPT-6 Astra: `7.13 Sol-eq`
- GPT-5.6 Terra: `5.59 Sol-eq`
- GPT-5.6 Sol: `3.57 Sol-eq`
- GPT-5.6 Luna: `3.32 Sol-eq`
- GPT-5.5: `2.93 Sol-eq`
- GPT-5.4: `2.07 Sol-eq`
- GPT-5.4 mini: `1.07 Sol-eq`
- Image generation: `1 Sol-eq`
- Other billable models not yet in the catalog: safe fallback `1 Sol-eq`.

Rules and UI behavior:

- Only successful billable requests (`HTTP 2xx`) reduce the pool.
- Failed requests and `/v1/models` remain zero-cost.
- `/proxy/` and a successful `/cekusage` lookup show all eight model estimates.
- Estimates are alternatives using the same remaining pool, **not values to add together**.
- Calculations use the exact remaining decimal value; a rounded display such as `559` may therefore yield a one-message difference from manually dividing the displayed rounded number.
- Existing historical `usage_events.sol_eq` values were not rewritten. The new weights apply to requests recorded after deployment.
- Model IDs currently exposed by 9Router were audited. Recognized aliases include `gcpr/gpt-5.6-terra`, `gcpr/gpt-5.6-sol`, Luna aliases, `gcpr/gpt-5.5`, `pecut-gpt-5.5`, `codex-gpt-5.5`, `gcpr/gpt-5.4`, and `gcpr/gpt-5.4-mini`.

Deployment evidence:

- Live source: `/home/ubuntu/managed-proxy/server.mjs`
- Pre-change backup: `/home/ubuntu/backups/acsa-model-cost-before-20260917-122804`
- Source checksum matched between staged and live copies.
- `node --check` passed.
- `acsa-managed-proxy.service`, `9router.service`, and `nginx` active.
- Public checks: `/proxy/` `200`, `/cekusage` `200`, unauthenticated `/v1/models` remains `401`.
- Authenticated `/proxy/` contains the new model-estimate panel and Sol-eq labels.
- Valid-token `/cekusage` contains the new effective-pool estimate panel.
- `/v1/models` authenticated smoke test passed and exposed 25 models.
- SQLite `PRAGMA quick_check` returned `ok`.
