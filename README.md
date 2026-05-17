# 9Router Colab with CLI Proxy

Dokumentasi aman untuk kolaborasi 9Router + managed proxy di `acsa.my.id`.

> Repo ini **tidak menyimpan password, API token, admin credential, database live, atau secret**.

## Ringkasan

- Domain publik: `https://acsa.my.id`
- 9Router dashboard/root: `https://acsa.my.id/dashboard` / `https://acsa.my.id/login`
- Managed OpenAI-compatible API: `https://acsa.my.id/v1/*`
- Managed admin dashboard: `https://acsa.my.id/proxy/`
- Public usage checker: `https://acsa.my.id/cekusage`

## Komponen

- 9Router upstream berjalan di server ACSA.
- Managed proxy berjalan di `127.0.0.1:8790` dan meneruskan request ke 9Router upstream.
- Nginx mengarahkan:
  - `/v1/*` ke managed proxy
  - `/proxy/` ke managed proxy admin UI
  - `/cekusage` ke managed proxy public checker
  - `/` ke 9Router

## Dokumen penting

- `docs/acsa-managed-proxy.md` — blueprint fitur managed proxy dan `/cekusage`.
- `docs/9router-server.md` — catatan mesin dan service 9Router ACSA.
- `runbooks/operations.md` — perintah operasional aman.
- `server-snapshots/` — snapshot konfigurasi non-secret dari server.

## Aturan keamanan

1. Jangan commit file `.env`, `admin.env`, SQLite DB live, token API, password SSH, atau credential lain.
2. Token dicek via hash; halaman `/cekusage` tidak boleh menampilkan daftar semua token.
3. Untuk lookup usage publik, token harus dikirim via POST body, bukan query string.
4. Backup sebelum patch file live.
5. Smoke test minimal setelah deploy: `/cekusage` 200, `/v1/models` tanpa token 401, service active, `nginx -t` ok.
