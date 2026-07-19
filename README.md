# docker-startup-kit

A self-hosted Docker Compose stack, reverse-proxied by Caddy with automatic
HTTPS and single sign-on. Use it two ways:

1. **A starter kit** to stand up your own home server: SSO, a wiki, remote
   desktop in the browser, document management, dynamic DNS, and bandwidth
   accounting.
2. **A reference layout** to replicate an existing stack onto a new machine.

Every service is one folder. Secrets never live in the repo. Bring services up
one at a time behind a single reverse proxy.

---

## What's in the box

| Service | What it does | Public host (example) | Behind SSO |
|---|---|---|---|
| [caddy](compose/caddy/README.md) | Reverse proxy + automatic HTTPS. Owns the shared network. | (routes all the below) | n/a |
| [authentik](compose/authentik/README.md) | Identity provider / single sign-on. | `mysso.mydomain.com` | is the IdP |
| [outline](compose/outline/README.md) | Team wiki / knowledge base. | `mywiki.mydomain.com` | via OIDC |
| [guacamole](compose/guacamole/README.md) | Clientless remote desktop (RDP/VNC/SSH in the browser). | `myremote.mydomain.com` | via OIDC |
| [paperless-ngx](compose/paperless-ngx/README.md) | Document scanning + OCR archive. | `mydocs.mydomain.com` | direct login |
| [ddclient](compose/ddclient/README.md) | Dynamic DNS updater (Porkbun). Keeps DNS pointed at your home IP. | n/a | n/a |
| [vnstat](compose/vnstat/README.md) | Bandwidth accounting for the host WAN interface. | n/a (CLI) | n/a |

Start with `caddy` + `authentik`. Add the rest as you need them.

---

## Repo layout

```
compose/<service>/
    docker-compose.yml     # the stack. Header comment documents local + public URL + start command.
    .env.example           # committed template. No secrets.
    README.md              # per-service setup + gotchas.
    Caddyfile              # caddy only: the reverse-proxy routing table.

.env_files/<service>.env   # YOUR real secrets. Git-ignored. You create these.
data/<service>/...         # persistent volumes (bind-mounted as ../../data/...). Git-ignored.
```

- Compose files reference volumes as `../../data/<service>/...` and pull secrets
  from `--env-file ../../.env_files/<service>.env`.
- `data/` and `.env_files/` are in [.gitignore](.gitignore). They are created on
  the host, never committed.

---

## Prerequisites

- A Linux host with **Docker Engine + Docker Compose v2** (`docker compose`, not
  `docker-compose`).
- **A domain** you control, with DNS you can edit. The examples use Porkbun (see
  [ddclient](compose/ddclient/README.md)); any provider works.
- **Ports 80 and 443** reachable from the internet (port-forward them on your
  router to this host). Caddy needs them to issue Let's Encrypt certificates.
- A **static public IP, or dynamic DNS** if your ISP IP changes (that is what
  ddclient is for).

---

## Quick start

```bash
# 1. Clone
git clone https://github.com/shaimak/docker-startup-kit.git
cd docker-startup-kit

# 2. Create the host-side folders the compose files expect
mkdir -p .env_files data

# 3. Bring up Caddy FIRST. It creates the shared proxy_network everything joins.
docker compose -f compose/caddy/docker-compose.yml up -d

# 4. For each service you want: copy the env template, fill in secrets,
#    add a Caddy site block, then start it.
cp compose/authentik/.env.example .env_files/authentik.env
#    ...edit .env_files/authentik.env (generate secrets, see the service README)...
docker compose -f compose/authentik/docker-compose.yml \
  --env-file .env_files/authentik.env up -d

# 5. Add the site block to compose/caddy/Caddyfile, then RECREATE Caddy
#    (a plain reload misses the change — see the gotcha below).
docker compose -f compose/caddy/docker-compose.yml up -d --force-recreate caddy
```

Each service README has its own env values, first-run steps, and quirks.

---

## How the networking fits together

- **Caddy owns one external network, `proxy_network`.** Its compose file creates
  it. Every service that needs a public URL joins `proxy_network` so Caddy can
  reach the container **by name** (e.g. `reverse_proxy outline-app:3000`).
- Multi-container services (anything with its own Postgres/Redis) also get a
  private `<service>_internal_network` so the database is not exposed to the
  proxy or to other services.
- Bring Caddy up before anything else, since it owns `proxy_network`. Everything
  else marks that network `external: true`.
- HTTPS is automatic: Caddy issues and renews Let's Encrypt certificates per
  hostname on first request, as long as the name resolves to your IP and 80/443
  are open.

---

## Conventions

- **Secrets live only in `.env_files/`.** Never in a compose folder, never in
  `data/`, never committed. The `.env.example` / `.env.sample` files are empty
  templates. Generate values with `openssl rand -hex 32` (or as the service
  README says).
- **`data/` holds live persistent volumes.** Back it up. Do not wipe it.
- **Single sign-on is optional per service.** Outline and Guacamole federate to
  Authentik over OIDC. Any other app can sit behind Authentik forward-auth using
  the template block in the [Caddyfile](compose/caddy/Caddyfile).

---

## Things to keep in mind

Editing the Caddyfile and running `caddy reload` reports *"config is unchanged"*
and keeps serving the old config. The Caddyfile is a single-file bind mount, and
most editors save by atomic-rename (new inode), so the running container keeps
the original. **Force-recreate instead:**

```bash
docker compose -f compose/caddy/docker-compose.yml up -d --force-recreate caddy
# verify the container actually sees the change:
docker exec caddy_reverse_proxy grep <newdomain> /etc/caddy/Caddyfile
```

---

## License

[MIT](LICENSE).
