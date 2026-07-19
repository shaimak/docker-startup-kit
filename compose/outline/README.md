# Outline — Team Wiki

A self-hosted wiki / knowledge base. Logs in through Authentik over OIDC, so it
has no separate user list.

- **Images:** `outlinewiki/outline` + `postgres:16-alpine` + `redis:alpine`
- **Containers:** `outline-app`, `outline-db`, `outline-redis`
- **Public host (example):** `mywiki.mydomain.com` → `outline-app:3000`
- **Storage:** local disk (`data/outline/storage`), no S3 needed
- **Networks:** `outline_internal_network` (db/redis) + `proxy_network` (app)

Depends on [authentik](../authentik/README.md) being up first.

---

## Setup

```bash
# 1. Env file
cp compose/outline/.env.example .env_files/outline.env
```

Fill in `.env_files/outline.env`:

- `OUTLINE_URL` = `https://mywiki.mydomain.com` (exact public URL)
- `OUTLINE_SECRET_KEY` = `openssl rand -hex 32`
- `OUTLINE_UTILS_SECRET` = `openssl rand -hex 32`
- `OUTLINE_DB_PASS` = `openssl rand -hex 24` (hex, so the DB URL never contains `/` or `+`)
- OIDC values (`OIDC_CLIENT_ID`, `OIDC_CLIENT_SECRET`, the three URIs) — see next
  section.

```bash
# 2. Start
docker compose -f compose/outline/docker-compose.yml \
  --env-file .env_files/outline.env up -d

# 3. Add the Caddy site block, then recreate Caddy
```

Caddyfile:

```caddy
mywiki.mydomain.com {
    encode zstd gzip
    reverse_proxy outline-app:3000
}
```

---

## OIDC provider (in Authentik)

1. In Authentik, create an **OAuth2/OpenID Provider**:
   - Redirect URI: `https://mywiki.mydomain.com/auth/oidc.callback`
   - Signing key: any certificate
2. Create an **Application** pointing at that provider. Note its slug.
3. Copy the credentials into `outline.env`:

```ini
OIDC_CLIENT_ID=<from Authentik>
OIDC_CLIENT_SECRET=<from Authentik>
OIDC_AUTH_URI=https://mysso.mydomain.com/application/o/authorize/
OIDC_TOKEN_URI=https://mysso.mydomain.com/application/o/token/
OIDC_USERINFO_URI=https://mysso.mydomain.com/application/o/userinfo/
OIDC_DISPLAY_NAME=SSO Login
```

4. Recreate `outline-app`. The login page now shows the SSO button.

---

## Notes

- `FORCE_HTTPS=true` is set because Outline sits behind Caddy. Internal
  plain-HTTP calls to `outline-app:3000` get redirected — always reach Outline
  through the public HTTPS URL.
- First user to log in via OIDC becomes the admin.
- Back up `data/outline/` (Postgres + uploaded files) to preserve the wiki.

```bash
docker compose -f compose/outline/docker-compose.yml \
  --env-file .env_files/outline.env up -d      # start / apply
docker logs -f outline-app                     # logs
```
