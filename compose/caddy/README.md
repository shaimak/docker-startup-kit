# Caddy — Reverse Proxy + Automatic HTTPS

The single entry point for the stack. Terminates HTTPS (automatic Let's
Encrypt), routes each public hostname to a container, and for some apps enforces
Authentik forward-auth. **Owns** the shared `proxy_network` that every published
service joins.

- **Image:** `caddy:latest`, container `caddy_reverse_proxy`
- **Ports:** 80, 443, 443/udp (HTTP/3)
- **Config:** `compose/caddy/Caddyfile` (single-file bind mount — see gotcha)
- **State/certs:** `data/caddy/data` (persistent — holds ACME account + certs; don't wipe)

Bring Caddy up **first**. Its compose file creates `proxy_network`; every other
service marks that network `external: true` and joins it.

---

## Start

```bash
docker compose -f compose/caddy/docker-compose.yml up -d
```

---

## A site block

A plain proxied site is three lines:

```caddy
wiki.example.com {
    encode zstd gzip
    reverse_proxy outline-app:3000
}
```

- The upstream is the target **container name** + its internal port. The
  container must be on `proxy_network`.
- HTTPS is automatic. Caddy issues the cert on first request via tls-alpn-01, as
  long as the hostname resolves to this host and 80/443 are reachable.

A site behind Authentik SSO wraps the upstream in a `route { ... forward_auth ... }`
block. See the commented template at the bottom of the
[Caddyfile](Caddyfile).

---

## ⚠️ Editing the Caddyfile = **recreate**, not reload

The Caddyfile is a **single-file** bind mount, so Docker pins the original inode.
Most editors save by atomic-rename (new inode), so the running container keeps
serving the **old** file and `caddy reload` reports *"config is unchanged."*

```bash
# After ANY Caddyfile edit:
docker compose -f compose/caddy/docker-compose.yml up -d --force-recreate caddy

# Verify the container actually sees the change:
docker exec caddy_reverse_proxy grep <newdomain> /etc/caddy/Caddyfile
```

A force-recreate is a ~2-3s blip across **all** sites.

---

## Adding a new public site

1. Point DNS at your home IP (an A record, or a CNAME to an existing anchor — see
   [ddclient](../ddclient/README.md)).
2. Add a site block to the Caddyfile.
3. Force-recreate Caddy (above).
4. The cert issues automatically on first request.

```bash
docker logs -f caddy_reverse_proxy | grep -iE '<domain>|certificate obtained|acme'
```

---

## Day-to-day

```bash
docker logs -f caddy_reverse_proxy                                              # access / TLS / ACME logs
docker exec caddy_reverse_proxy caddy validate --config /etc/caddy/Caddyfile    # syntax check
docker compose -f compose/caddy/docker-compose.yml up -d --force-recreate caddy # apply edits
```

> Certs live in `data/caddy/data/caddy/certificates/...` and auto-renew ~30 days
> before expiry. Back up `data/caddy/` to keep the ACME account + certs across a
> rebuild.
