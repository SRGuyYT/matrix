---

# 🚀 Sky0Cloud Matrix/Synapse "God-Mode" Deployment Guide

This guide overhauls the deployment for `sky0cloud.dpdns.org`. It is optimized for **Cloudflare Tunnels**, fixes **CORS loops**, and ensures **PostgreSQL** stability.

## 1) High-Performance Stack Deployment

Before running, ensure your `docker-compose.yml` has `shm_size: '256mb'` for Postgres to prevent database lag.

```bash
docker compose pull
docker compose up -d

```

**Check Health:**

```bash
# Ensure postgres shows "Healthy" before synapse starts
docker compose ps

```

## 2) Caddy "Clean-Header" Model

To fix the **"Multiple Access-Control-Allow-Origin"** error, Caddy must strip headers from Synapse and set its own.

**Update your Caddyfile with this logic:**

```caddy
sky0cloud.dpdns.org {
    # Proxy Matrix API & Synapse Admin API
    reverse_proxy /_matrix/* /_synapse/* http://sky0cloud-synapse:8008 {
        header_up X-Forwarded-For {header.Cf-Connecting-Ip}
        # Strip duplicate CORS from Synapse
        header_down -Access-Control-Allow-Origin
        header_down Access-Control-Allow-Origin "https://sky0cloud.dpdns.org"
    }

    # Serve Element Web
    reverse_proxy * http://sky0cloud-element-web:80
}

matrix.sky0cloud.dpdns.org {
    reverse_proxy * http://sky0cloud-synapse-admin:80
}

```

## 3) Synapse "Trust" Configuration

The `#1 reason` for email link failure is Synapse not trusting the proxy. Ensure these keys exist in `homeserver.yaml`:

```yaml
public_baseurl: "https://sky0cloud.dpdns.org/"
trusted_proxies:
  - "127.0.0.1"
  - "172.16.0.0/12" # Docker Bridge
  - "10.0.0.0/8"    # Cloudflare Tunnel Internal

account_threepid:
  email:
    client_can_lookup: true
    address_is_trusted: true # Stops the "No validated 3pid session" error

```

## 4) The "Zero-Downtime" Postgres Migration

Instead of manual snapshots, use the **Docker Stream** method. It’s faster and less prone to file permission errors.

### Step A: Prepare the Postgres Config

1. Create `homeserver-postgres.yaml` with your `psycopg2` settings.
2. Ensure the `database:` section points to `host: sky0cloud-postgres`.

### Step B: Run the Port Script

Stop Synapse so the database is "quiet," then run the migration:

```bash
docker compose stop synapse

docker compose run --rm synapse python3 -m synapse.app.port_db \
  --sqlite-database /data/homeserver.db \
  --postgres-config /data/homeserver-postgres.yaml

```

### Step C: The Cutover

```bash
mv synapse/homeserver.yaml synapse/homeserver.yaml.bak
cp synapse/homeserver-postgres.yaml synapse/homeserver.yaml
docker compose up -d synapse

```

## 5) Validation Checklist (The "Success" Test)

Run these commands. If any return a `400` or `502`, the Tunnel is misconfigured.

| Test | Command | Expected Result |
| --- | --- | --- |
| **Federation** | `curl https://sky0cloud.dpdns.org/_matrix/client/versions` | `{"versions": [...` |
| **Admin API** | `curl https://sky0cloud.dpdns.org/_synapse/admin/v1/server_version` | `{"server_version": ...` |
| **CORS Check** | `curl -I https://sky0cloud.dpdns.org/_matrix/client/versions` | Only ONE `Access-Control-Allow-Origin` |

## 6) Security Hardening

After your first login, perform these three actions immediately:

1. **Disable Public Registration:** Set `enable_registration: false` in `homeserver.yaml`.
2. **Rotate Secrets:** If you ever committed your `homeserver.yaml` to GitHub, change your `macaroon_secret_key` and `registration_shared_secret` immediately.
3. **Cloudflare Access:** In the Cloudflare dashboard, put `matrix.sky0cloud.dpdns.org` (the Admin UI) behind a **One-Time Pin (OTP)** email wall so only you can even see the login page.

---

**Pro Tip:** If you see `psycopg2.OperationalError: FATAL: password authentication failed`, check your `docker-compose.yml` to ensure the Postgres password matches the one in `homeserver.yaml` exactly. Synapse is very picky about special characters in passwords!
