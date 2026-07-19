# Paperless-ngx — Document Archive

Scan, OCR, index, and full-text-search your documents. It has its own login, so
it does not need Authentik (you can still put it behind forward-auth if you
want).

- **Images:** `paperless-ngx/paperless-ngx` + `postgres:17` + `redis:7`
- **Containers:** `paperless-webserver`, `paperless-db`, `paperless-broker`
- **Public host (example):** `mydocs.mydomain.com` → `paperless-webserver:8000`
- **Networks:** `paperless_internal_network` (db/redis) + `proxy_network` (web)

This service also keeps a direct host port (`PAPERLESS_PORT_HOST`, default 8000)
for local access alongside the Caddy route.

---

## Setup

```bash
# 1. Env file
cp compose/paperless-ngx/.env.example .env_files/paperless-ngx.env
```

Fill in `.env_files/paperless-ngx.env`:

- `PAPERLESS_SECRET_KEY` = `openssl rand -hex 32`
- `PAPERLESS_ADMIN_PASSWORD` = your admin password
- `PAPERLESS_DBPASS` = `openssl rand -hex 24` (hex, so the DB password never contains `/` or `+`)
- `USERMAP_UID` / `USERMAP_GID` = your host user's `id -u` / `id -g` (matters for
  file permissions on the consume/media folders)
- `PAPERLESS_URL` = `https://mydocs.mydomain.com`

```bash
# 2. Start
docker compose -f compose/paperless-ngx/docker-compose.yml \
  --env-file .env_files/paperless-ngx.env up -d

# 3. Add the Caddy site block, then recreate Caddy
```

Caddyfile:

```caddy
mydocs.mydomain.com {
    encode zstd gzip
    reverse_proxy paperless-webserver:8000
}
```

---

## Using it

- Log in as `PAPERLESS_ADMIN_USER` with the password you set.
- Drop files into `data/paperless-ngx/consume/` and Paperless ingests and OCRs
  them automatically.
- Set `PAPERLESS_OCR_LANGUAGE` (default `eng`) to your document language.
- Back up `data/paperless-ngx/` (Postgres + media + originals).
