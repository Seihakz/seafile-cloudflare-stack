# Seafile 13 + SeaDoc on Docker behind a Cloudflare Tunnel

Production deployment template for a self-hosted [Seafile](https://www.seafile.com) 13 instance
with collaborative document editing ([SeaDoc](https://github.com/haiwen/seafile-docs)) and
Cloudflare Zero Trust edge networking — **zero open inbound ports required**.

All traffic flows through a Cloudflare Tunnel (`cloudflared`), so the VPS exposes nothing but SSH.
TLS is terminated at the Cloudflare edge; internally, a `caddy-docker-proxy` instance routes
requests to the Seafile and SeaDoc containers using Docker labels.

> **Status:** Battle-tested in production. Compose files are fully parameterized — every secret
> is injected via a single `.env` file that is never committed.

---

## Architecture

```
                    ┌──────────────────────────────────────────────┐
                    │                Cloudflare Edge               │
                    │   seafile.yourdomain.com  (proxied DNS, TLS) │
                    └──────────────────────┬───────────────────────┘
                                           │ outbound-only tunnel (QUIC/443)
                                           ▼
┌────────────────────────────── VPS (Docker: seafile-net) ──────────────────────────────┐
│                                                                                        │
│   ┌────────────┐   http://seafile-caddy:80   ┌──────────────────────────────────┐     │
│   │ cloudflared │───────────────────────────▶│ seafile-caddy (caddy-docker-proxy)│    │
│   └────────────┘                             └───────────┬───────────┬──────────┘     │
│                                                          │           │                │
│              default: /*                    /socket.io/*, /sdoc-server/*             │
│                                                          ▼           ▼                │
│                                                   ┌──────────┐ ┌──────────┐          │
│                                                   │  seafile │ │  seadoc  │          │
│                                                   │ (13.0-mc)│ │(sdoc 2.0)│          │
│                                                   └────┬─────┘ └────┬─────┘          │
│                                                        │            │                │
│                                          ┌─────────────┴────────────┘                │
│                                          ▼                                             │
│                              ┌──────────────┐    ┌──────────────┐                     │
│                              │ seafile-mysql│    │ seafile-redis│                     │
│                              │ (mariadb 10.11)   │              │                     │
│                              └──────────────┘    └──────────────┘                     │
└────────────────────────────────────────────────────────────────────────────────────────┘

Persistent volumes:  /opt/seafile-data   (libraries, conf, avatars)
                     /opt/seafile-mysql  (database)
                     /opt/seadoc-data    (SeaDoc)
                     /opt/seafile-caddy  (Caddy state + certs)
```

### Container inventory

| Container        | Image                                          | Role                                                            |
| ---------------- | ---------------------------------------------- | --------------------------------------------------------------- |
| `seafile`        | `seafileltd/seafile-mc:13.0-latest`            | Seafile server + Seahub web UI + Go fileserver                  |
| `seadoc`         | `seafileltd/sdoc-server:2.0-latest`            | Collaborative `.sdoc` document editor backend                   |
| `seafile-mysql`  | `mariadb:10.11`                                | Database (ccnet_db, seafile_db, seahub_db) with auto-upgrade    |
| `seafile-redis`  | `redis`                                        | Cache/session store (password-protected, non-persistent)        |
| `seafile-caddy`  | `lucaslorentz/caddy-docker-proxy:2.12-alpine`  | Label-driven reverse proxy / internal router                    |
| `cloudflared`    | `cloudflare/cloudflared:latest`                | Cloudflare Tunnel connector (token-authenticated, remote-managed)|

### Request routing

1. **Cloudflare edge** terminates TLS for `seafile.yourdomain.com` (DNS record proxied through Cloudflare).
2. **Tunnel** delivers requests to the `cloudflared` container over an outbound connection — no listener needed.
3. **Tunnel ingress rule** (managed in the Cloudflare dashboard, *remotely-managed* tunnel) sends everything to `http://seafile-caddy:80`.
4. **Caddy** routes by Docker label:
   - `/*` → `seafile:80`
   - `/socket.io/*` → `seadoc:80` (rewritten to `/socket.io{uri}`)
   - `/sdoc-server/*` → `seadoc:80`

This mirrors the stock Seafile ingress rules as Caddy labels, so the SeaDoc websocket works
without hand-written proxy configuration.

---

## Prerequisites

- A Linux server (Ubuntu 22.04/24.04 tested) with Docker Engine + Docker Compose v2
- A domain managed by Cloudflare (free plan is sufficient)
- `openssl` for generating the JWT private key

---

## Setup

### 1. Create a Cloudflare Tunnel

1. Cloudflare dashboard → **Zero Trust → Networks → Tunnels → Create a tunnel → Cloudflared**.
2. Name it (e.g. `seafile-vps`). Copy the generated **tunnel token**.
3. Add a **public hostname**:
   - Subdomain: `seafile`, Domain: `yourdomain.com`
   - Service: `HTTP` → `seafile-caddy:80`
4. Create a proxied DNS record for `seafile.yourdomain.com` (the tunnel wizard does this).

The tunnel is *remotely managed* — its ingress rules live in the dashboard, not in a local
`config.yml`. No `cert.pem` or tunnel credential JSON files ever touch the server.

### 2. Configure the environment

```bash
sudo mkdir -p /opt/seafile && cd /opt/seafile
# copy seafile-server.yml, seadoc.yml, caddy.yml, .env.example from this repo
sudo cp .env.example .env
sudo nano .env
```

Generate the required secrets:

```bash
# JWT private key (used by Seafile, SeaDoc, and the metadata server)
openssl genrsa 2048 2>/dev/null | openssl pkcs8 -topk8 -nocrypt

# Strong passwords — use a DIFFERENT random value for each variable:
openssl rand -base64 24
```

Fill in at minimum:

| Variable                        | Notes                                                       |
| ------------------------------- | ----------------------------------------------------------- |
| `SEAFILE_SERVER_HOSTNAME`       | `seafile.yourdomain.com`                                    |
| `SEAFILE_MYSQL_DB_PASSWORD`     | Seafile DB user password                                    |
| `INIT_SEAFILE_MYSQL_ROOT_PASSWORD` | MariaDB root password (used on first boot only)          |
| `REDIS_PASSWORD`                | Redis auth password                                         |
| `JWT_PRIVATE_KEY`               | The full multi-line PEM key, quoted                         |
| `INIT_SEAFILE_ADMIN_EMAIL`      | Admin account created on first boot                         |
| `INIT_SEAFILE_ADMIN_PASSWORD`   | Admin password on first boot (rotate afterwards)            |
| `CLOUDFLARE_TUNNEL_TOKEN`       | Token from step 1                                           |

### 3. Deploy

```bash
cd /opt/seafile
sudo docker compose up -d
sudo docker compose ps        # wait for db + seafile to report healthy
```

First boot initializes the databases and creates the admin account (2–5 minutes).
Then open `https://seafile.yourdomain.com`.

---

## Configuration reference

`seafile-server.yml`, `seadoc.yml`, and `caddy.yml` are merged by Compose via the
`COMPOSE_FILE` variable in `.env`:

```dotenv
COMPOSE_FILE=seafile-server.yml,caddy.yml,seadoc.yml
COMPOSE_PATH_SEPARATOR=,
```

Optional toggles in `.env` (all have sane defaults in the compose files):

- `ENABLE_SEADOC=true` — SeaDoc collaborative editor
- `ENABLE_NOTIFICATION_SERVER=false` — Seafile notification server
- `ENABLE_SEAFILE_AI=false` / `SEAFILE_AI_LLM_*` — Seafile AI assistant
- `ENABLE_GO_FILESERVER=true`, `NON_ROOT=false`, `TIME_ZONE`, `SITE_ROOT`

---

## Upgrades

```bash
cd /opt/seafile
sudo docker compose pull
sudo docker compose up -d
```

`MARIADB_AUTO_UPGRADE=1` handles database schema upgrades automatically.
**Snapshot/backup the data directories first** (see below).

---

## Backup & Disaster Recovery

### What to back up

| Data                              | Contents                          | Method                       |
| --------------------------------- | --------------------------------- | ---------------------------- |
| `/opt/seafile-mysql/db`           | All Seafile databases             | `mariadb-dump` (logical)     |
| `/opt/seafile-data/seafile/conf`  | seahub_settings.py, SECRET_KEY, JWT keys | file copy             |
| `/opt/seafile-data/seafile/data`  | File libraries (content-addressed)| file copy / rsync / restic   |
| `/opt/seafile/.env`               | All credentials                   | encrypted copy only          |

### Logical DB dump (recommended over copying raw InnoDB files)

```bash
sudo docker exec seafile-mysql sh -c \
  'mariadb-dump -uroot -p"$MARIADB_ROOT_PASSWORD" --all-databases --single-transaction' \
  > seafile-db-$(date +%F).sql
```

### Full restore to a fresh VPS

1. Provision a new server, install Docker, follow **Setup** steps 1–2 (reuse the same
   `JWT_PRIVATE_KEY` and `SEAFILE_MYSQL_DB_PASSWORD` from the backed-up `.env`).
2. Stop the stack: `sudo docker compose down`.
3. Restore `/opt/seafile-data/seafile/conf` (contains `SECRET_KEY` — **mandatory**,
   libraries cannot be decrypted without it).
4. Restore library data into `/opt/seafile-data/seafile/data`.
5. Start only the DB: `sudo docker compose up -d db`.
6. Load the dump: `sudo docker exec -i seafile-mysql sh -c \
   'mariadb -uroot -p"$MARIADB_ROOT_PASSWORD"' < seafile-db-<date>.sql`
7. Start everything: `sudo docker compose up -d`.

> **Critical:** the `SECRET_KEY` inside `/opt/seafile-data/seafile/conf/seahub_settings.py`
> and the `JWT_PRIVATE_KEY` are irreplaceable. Losing either means losing the data.

---

## Security model

- **No inbound ports.** The tunnel is outbound-only; the host firewall can deny 80/443.
- **Single secret surface.** Everything sensitive lives in one `600`-mode `.env`, excluded by `.gitignore`.
- **Remotely-managed tunnel.** No `cert.pem` / `<tunnel-id>.json` credentials on disk — the
  token is scoped to this tunnel only and can be revoked from the dashboard.
- **Network isolation.** Containers communicate only on the `seafile-net` bridge; DB and Redis
  publish no host ports.

### Hardening checklist

- [ ] Use a **unique** password for each of: MariaDB root, Seafile DB user, Redis, admin account
- [ ] Set Cloudflare SSL/TLS mode to **Full (strict)** or rely on tunnel-only origin
- [ ] Enable Cloudflare Access (Zero Trust policy) in front of the web UI for MFA
- [ ] Restrict the host firewall to SSH only (`ufw allow 22; ufw enable`)
- [ ] Rotate `INIT_SEAFILE_ADMIN_PASSWORD` after first login
- [ ] Pin container image digests for reproducibility

---

## Troubleshooting

| Symptom | Fix |
| --- | --- |
| 502 from Cloudflare | `docker logs cloudflared`, check ingress service name `seafile-caddy:80` |
| SeaDoc editor won't connect | Verify `/socket.io/*` and `/sdoc-server/*` Caddy labels on `seadoc`: `docker inspect seadoc \| grep -A20 Labels` |
| Seafile unhealthy | `docker logs seafile -f`; usually a bad DB password or missing `JWT_PRIVATE_KEY` |
| Forgot admin password | `docker exec -it seafile reset-admin.sh` |
| Caddy routing stale | `docker restart seafile-caddy` (it re-reads labels) |

---

## License & attribution

Configuration template released under MIT. Seafile is licensed by Seafile Ltd.;
Caddy by Matthew Holt & contributors; cloudflared by Cloudflare Inc.
