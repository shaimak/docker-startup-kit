# Authentik — Single Sign-On (Identity Provider)

The identity provider for the stack. Other apps (Outline, Guacamole, anything
behind forward-auth) federate to it over OIDC. Nothing else needs its own user
database.

- **Images:** `ghcr.io/goauthentik/server` (server + worker) + `postgres:16-alpine` + `redis:alpine`
- **Containers:** `authentik-server`, `authentik-worker`, `authentik-db`, `authentik-redis`
- **Public host (example):** `sso.example.com` → `authentik-server:9000`
- **Networks:** `authentik_internal_network` (db/redis) + `proxy_network` (server)

---

## Setup

```bash
# 1. Env file
cp compose/authentik/.env.example .env_files/authentik.env

# 2. Fill in secrets:
#    AUTHENTIK_SECRET_KEY  ->  openssl rand -hex 32
#    PG_PASS               ->  openssl rand -hex 24
#    AUTHENTIK_URL         ->  https://sso.example.com  (your public URL — used for redirects)

# 3. Start the stack
docker compose -f compose/authentik/docker-compose.yml \
  --env-file .env_files/authentik.env up -d

# 4. Add the Caddy site block (below), then recreate Caddy
```

Caddyfile:

```caddy
sso.example.com {
    encode zstd gzip
    reverse_proxy authentik-server:9000
}
```

---

## First run

1. Browse to `https://sso.example.com/if/flow/initial-setup/`.
2. Set the `akadmin` password. This is your break-glass admin — keep it safe.
3. From here, create Providers + Applications for each app you want to protect.

---

## Wiring another app to it

- **OIDC apps (Outline, etc.):** create an *OAuth2/OpenID Provider* + *Application*
  in Authentik, then paste the client id/secret and the authorize/token/userinfo
  URLs into that app's env file. See [outline](../outline/README.md).
- **Apps with no auth of their own:** protect them with Caddy **forward-auth**.
  Create a *Proxy Provider* (mode `forward_single`, external host = the app's
  URL) + Application, attach it to the **embedded outpost**, then use the
  forward-auth template block in the [Caddyfile](../caddy/Caddyfile).

The standard Authentik endpoints for OIDC apps:

```
authorize:  https://sso.example.com/application/o/authorize/
token:      https://sso.example.com/application/o/token/
userinfo:   https://sso.example.com/application/o/userinfo/
jwks:       https://sso.example.com/application/o/<app-slug>/jwks/
issuer:     https://sso.example.com/application/o/<app-slug>/
```

> Server-to-server calls (like a JWKS fetch from another container) can target
> `http://authentik-server:9000/...` internally to avoid NAT-hairpin issues,
> while the browser-facing URLs stay on the public `sso.example.com`.

---

## Day-to-day

```bash
EF=.env_files/authentik.env
docker compose -f compose/authentik/docker-compose.yml --env-file $EF up -d   # start / apply
docker logs -f authentik-server                                               # logs

# Scripting Authentik objects (providers/apps) from the API shell:
docker exec authentik-server ak shell
# ak shell chokes on piped multi-line input — write a script to a file and exec() it:
#   docker exec authentik-server ak shell -c "exec(open('/tmp/script.py').read())"
```

> Back up `data/authentik/` (Postgres + media + certs) to preserve users,
> providers, and signing keys across a rebuild.
