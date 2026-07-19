# docker-startup-kit

A self-hosted Docker Compose stack, reverse-proxied by Caddy with automatic
HTTPS and single sign-on. Use it two ways:

1. **A starter kit** to stand up your own home server: SSO, a wiki, remote
   desktop in the browser, document management, dynamic DNS, and bandwidth
   accounting.
2. **A reference layout** to replicate an existing stack onto a new machine.

Every service is one folder. Secrets never live in the repo. Bring services up
one at a time behind a single reverse proxy.

I run this on **Debian**, but the structure works out of the box on any Linux.

---

## Who this is for

- Anyone who wants to run their own services instead of renting SaaS.
- People who care about privacy and owning their own data.
- Homelab learners who want a real, working stack to study and extend.
- Small teams and businesses. The same layout scales to a powerful machine with
  an account for every employee (see **Enterprise-ready** below).

You do not need to be a sysadmin. You need one always-on machine, a domain, and
a willingness to copy-paste commands.

---

## Why self-host

- **You own the data.** It lives on your disk, not on someone else's server.
- **It is always within physical reach.** If something breaks you can walk over
  to the machine. If you stop paying a SaaS bill, nobody locks you out.
- **Privacy.** Your documents, notes, and desktop sessions never leave your
  hardware.
- **It is a good exercise.** You learn networking, DNS, reverse proxies, TLS,
  containers, and SSO by running the real thing.

## Why Docker

- **Lift and shift.** The whole stack is portable. To move to a new machine you
  copy three things: this repo, your `data/` folder, and your `.env_files/`
  folder. Bring the containers up on the new box and everything is exactly where
  you left it. No reinstalling, no reconfiguring.
- **Reproducible.** Each service is pinned in a compose file. The same command
  brings up the same stack every time.
- **Isolated.** Every service and its database run in their own containers on
  their own network. One service cannot see another's internals.
- **Easy to set up.** No compiling, no dependency hell. `docker compose up -d`
  and the service is running.

## Single sign-on with Authentik

[Authentik](compose/authentik/README.md) is the identity provider for the stack.
Log in once and every connected service recognises you. No separate username and
password per app.

- One account for the wiki, remote desktop, and anything else you put behind it.
- Apps that speak OIDC (Outline, Guacamole) federate to Authentik directly.
- Apps that have no login of their own can still sit behind Authentik using
  Caddy forward-auth (template in the [Caddyfile](compose/caddy/Caddyfile)).

## Enterprise-ready

This is the same pattern a company would use. Run it on a powerful machine,
create an Authentik account for each employee, and every internal tool is behind
one login. Add or remove a person in one place and their access to every service
follows. Data stays on hardware you control.

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

---

## Requirements

- **One machine, ideally on 24/7.** Anything from a spare laptop to a mini PC to
  a server works. It stays on so your services stay reachable. I run Debian; any
  Linux is fine.
- **Docker Engine + Docker Compose v2** (`docker compose`, not the old
  `docker-compose`).
- **A domain**, with DNS you can edit.
  - I bought mine on **Porkbun**. You can buy from any registrar (Cloudflare,
    Namecheap, etc).
  - **Free alternatives** exist that give you a subdomain instead of a full
    domain: **DuckDNS** (`yourname.duckdns.org`) and **No-IP**
    (`yourname.ddns.net`). These also handle the dynamic-IP updates themselves,
    so you can skip the Porkbun ddclient piece.
- **Ports 80 and 443 reachable from the internet.** Port-forward them on your
  router to this machine. Caddy needs them to get Let's Encrypt certificates.
- **A static public IP, or dynamic DNS.** Most home connections change IP over
  time. [ddclient](compose/ddclient/README.md) keeps your DNS pointed at the
  current IP. (A free DDNS provider like DuckDNS does the same.)

---

## Folder structure

Everything lives under one parent folder. I use `~/apps/`. You can rename it or
move the whole set anywhere.

```
apps/
├── compose/<service>/
│   ├── docker-compose.yml   # the stack. Header comment has the local + public URL + start command.
│   ├── .env.example         # committed template. No real secrets.
│   ├── README.md            # per-service setup + gotchas.
│   └── Caddyfile            # caddy only: the reverse-proxy routing table.
│
├── data/<service>/...       # all persistent data (databases, files). Bind-mounted as ../../data/...
│
└── .env_files/<service>.env # your REAL secrets, one file per service.
```

**About the env files.** Normally Docker reads a `.env` file sitting next to the
compose file. I do it differently: I keep all the real env files together in one
`.env_files/` folder and point each service at its file with `--env-file`. Two
reasons:

- All my secrets are in one place, easy to back up and easy to keep private.
- The per-service folders stay clean and safe to share. `.env_files/` is
  git-ignored, so secrets never land in the repo. That is also why this folder
  is **not** included here: it holds my actual credentials.

**What you do when cloning this repo:** create your own `.env_files/` folder,
copy each service's `.env.example` (or `.env.sample`) into it as
`<service>.env`, and fill in your own values.

```bash
mkdir -p .env_files
cp compose/outline/.env.example .env_files/outline.env
# then edit .env_files/outline.env with your own secrets
```

If you prefer the conventional layout, you can instead drop a `.env` next to each
compose file and leave off `--env-file`. Both work.

`data/` and `.env_files/` are both in [.gitignore](.gitignore). They live on the
host and are never committed.

---

## Generating secrets

Several services need a strong password or secret key. Generate them with
`openssl` in the terminal:

```bash
openssl rand -hex 32     # 64-char secret. Use for app secret keys.
openssl rand -hex 24     # 48-char secret. Good for database passwords.
```

- `openssl rand -hex N` outputs N bytes as hex (characters `0-9` and `a-f`), so
  the result is `2 x N` characters long.
- **Use hex, not base64, for database passwords.** base64 can include `/` and
  `+`, which some databases and connection strings mishandle. Hex is always safe.

Each service README lists exactly which values to generate.

---

## Setup order

Bring the pieces up in this order. Each service's own README has the detailed
steps (env values, first-run, creating your account, wiring SSO).

1. **[caddy](compose/caddy/README.md)**: start it first. It creates the shared
   `proxy_network` that every other service joins.
2. **[authentik](compose/authentik/README.md)**: start it next, then open its
   URL and create your admin account. This is your single sign-on. Every other
   service authenticates against it.
3. **The services you want**: [outline](compose/outline/README.md),
   [guacamole](compose/guacamole/README.md),
   [paperless-ngx](compose/paperless-ngx/README.md), and so on. For the SSO ones
   you create an OIDC provider in Authentik (the per-service README shows how),
   paste the credentials into that service's env file, and start it.
4. **Supporting pieces**: [ddclient](compose/ddclient/README.md) for dynamic
   DNS and [vnstat](compose/vnstat/README.md) for bandwidth, any time.

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

# 4. For each service: copy the env template, fill in secrets (see "Generating
#    secrets"), add a Caddy site block, then start it.
cp compose/authentik/.env.example .env_files/authentik.env
#    ...edit .env_files/authentik.env ...
docker compose -f compose/authentik/docker-compose.yml \
  --env-file .env_files/authentik.env pull
docker compose -f compose/authentik/docker-compose.yml \
  --env-file .env_files/authentik.env up -d

# 5. Add the site block to compose/caddy/Caddyfile, then RECREATE Caddy
#    (a plain reload misses the change, see "Things to keep in mind").
docker compose -f compose/caddy/docker-compose.yml up -d --force-recreate caddy
```

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
- HTTPS is automatic. Caddy issues and renews Let's Encrypt certificates per
  hostname on first request, as long as the name resolves to your IP and 80/443
  are open.

---

## Conventions

- **Secrets live only in `.env_files/`.** Never in a compose folder, never in
  `data/`, never committed.
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
