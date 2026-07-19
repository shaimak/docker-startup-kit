# ddclient — Dynamic DNS (Porkbun)

Keeps your home WAN IP current in DNS so the stack stays reachable as your ISP
changes the IP. Runs as a small container and talks to the Porkbun API. Any
ddclient-supported provider works; this doc uses Porkbun because its apex/ALIAS
behaviour has sharp edges worth documenting.

- **Provider:** Porkbun (registrar + DNS)
- **Updater:** `lscr.io/linuxserver/ddclient`, container `ddclient`
- **Checks IP every:** 300s (`daemon=300`), via `ifconfig.me/ip`

> ⚠️ **`data/ddclient/config/ddclient.conf` holds your real Porkbun API keys**
> (`pk1_...` + `sk1_...`), account-wide. Keep it private. This is the one service
> that deviates from the `.env_files/` convention — the linuxserver image reads
> the keys from the conf file, not env vars.

---

## DNS topology — one A record, everything else points at it

ddclient only updates **subdomain A records**; the apex and all service
hostnames follow via CNAME/ALIAS. So a changing IP only needs the one anchor
refreshed.

```
example.com               A      -> wherever your main site lives (need NOT be this host)
root.example.com          A      -> <home WAN IP>     ← ddclient updates this
  sso / wiki / control . example.com   CNAME -> root.example.com

# Optional second domain for an anonymous front-end (see guacamole README):
fridayrun.example         ALIAS  -> root.fridayrun.example   (apex follows the anchor)
root.fridayrun.example    A      -> <home WAN IP>     ← ddclient updates this
```

Give a neutral front-end domain its **own** anchor rather than a CNAME to
`root.example.com` — a CNAME would leak the main name in a `dig`. Point its apex
at its own anchor with a Porkbun **ALIAS**.

---

## `data/ddclient/config/ddclient.conf`

```ini
daemon=300
syslog=yes
pid=/config/ddclient.pid
cache=/config/ddclient.cache          # NOTE: ignored by the image — see gotcha #2
ssl=yes

# --- Porkbun ---
use=web, web=ifconfig.me/ip
protocol=porkbun
apikey=pk1_...                        # account-wide (kept in this file)
secretapikey=sk1_...
root.example.com                      # anchor for *.example.com
root.fridayrun.example                # optional second anchor
```

`compose/ddclient/docker-compose.yml` just mounts `../../data/ddclient/config:/config`
and runs the image (`restart: unless-stopped`).

---

## Setup

```bash
mkdir -p data/ddclient/config
# create data/ddclient/config/ddclient.conf (above) with your Porkbun keys + anchors
cp compose/ddclient/.env.example .env_files/ddclient.env   # PUID/PGID/TZ only
docker compose -f compose/ddclient/docker-compose.yml \
  --env-file .env_files/ddclient.env up -d
docker logs -f ddclient
```

---

## Gotchas (the things that actually bit us)

1. **ddclient can NOT update a bare apex on Porkbun — it returns `400`.** Every
   record it manages must be a **subdomain** (`root.example.com` works; a bare
   `example.com` fails every cycle with `FAILED: [porkbun][example.com]> 400`).
   Fix = update a subdomain anchor and point the apex at it with a **Porkbun
   ALIAS** (apex CNAMEs aren't valid DNS; ALIAS is Porkbun's apex-flattening
   equivalent and returns just the A in a lookup, with no name leak).

2. **The real cache is `/run/ddclient-cache/ddclient.cache`, not
   `/config/ddclient.cache`.** The image overrides the `cache=` setting, so
   editing/deleting the `/config` cache does nothing. After a failed update
   ddclient enforces a **5-minute per-host retry cooldown**
   (`skipped ... less than 5 minutes since the previous attempt which failed`).
   Using a *new* hostname sidesteps the cooldown (fresh host = no history).

3. **Porkbun requires per-domain API-access opt-in.** A newly added domain
   returns `"Domain is not opted in to API access"` until you enable it at
   porkbun.com → **Account → API Access** (global) or the per-domain **API
   ACCESS** switch in Domain Management. The same account-wide key then works.

4. **New Porkbun domains ship with parking records.** A default apex
   `ALIAS → pixie.porkbun.com` and a `*` CNAME. Delete both before adding your
   own records (the apex ALIAS conflicts with an apex A/ALIAS you add).

5. **`restart: unless-stopped` matters.** A container that gets removed stops
   updating DNS silently — you only notice when the IP changes. The restart
   policy brings it back on reboot.

---

## Day-to-day

```bash
EF=.env_files/ddclient.env
docker compose -f compose/ddclient/docker-compose.yml --env-file $EF up -d   # start / apply
docker restart ddclient                                                      # re-read ddclient.conf
docker logs -f ddclient                                                      # watch update cycles

# Healthy output looks like:
#   SUCCESS: [porkbun][root.example.com]> skipped: IPv4 address was already set to <ip>

# Is DNS actually in sync?
dig +short root.example.com @1.1.1.1     # vs:
curl -s ifconfig.me/ip
```

### Add a new dynamic domain

1. Buy it in Porkbun; enable **API ACCESS** for it (gotcha #3).
2. Delete the default parking records (gotcha #4).
3. Create a subdomain A anchor `root.<domain>` → current IP, and (if you want the
   apex live) an apex `ALIAS → root.<domain>`.
4. Add `root.<domain>` to `ddclient.conf`; `docker restart ddclient`.
5. Add the Caddy site block + recreate Caddy ([caddy README](../caddy/README.md)) —
   the cert issues automatically.

> You can also make Porkbun DNS edits directly from the API with the keys in
> `ddclient.conf`, e.g.
> `POST https://api.porkbun.com/api/json/v3/dns/{retrieve,create,delete}/<domain>`
> (`ping` validates keys). ddclient itself only ever updates the A records listed
> in the conf.
