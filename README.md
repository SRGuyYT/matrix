---

# 🚀 Sky0Cloud Matrix/Synapse Deployment Guide

This guide overhauls the deployment for `sky0cloud.dpdns.org`. It is optimized for **Cloudflare Tunnels**, fixes **CORS loops**, and ensures **PostgreSQL** stability.

---

## 1) High-Performance Stack Deployment

Before running, ensure your `docker-compose.yml` has `shm_size: '256mb'` for Postgres to prevent database lag. Also, ensure **Element Web is mapped to host port 8090**.

```bash
docker compose pull
docker compose up -d
```

**Check Health:**

```bash
# Ensure postgres shows "Healthy" before Synapse starts
docker compose ps
```

---

## 2) Caddy "Clean-Header" Model

To fix the **"Multiple Access-Control-Allow-Origin"** error, Caddy must strip headers from Synapse and set its own.

**Update your Caddyfile with this logic:**

```caddy id="mijwna"
{
    email chapmlinks@gmail.com
    auto_https off
}

# Synapse Admin UI
matrix.sky0cloud.dpdns.org:80 {
    reverse_proxy sky0cloud-synapse-admin:80
}

# Main domain (Element + Matrix API)
sky0cloud.dpdns.org:80 {

    # Global CORS
    header {
        Access-Control-Allow-Origin *
        Access-Control-Allow-Methods "GET, POST, PUT, DELETE, OPTIONS"
        Access-Control-Allow-Headers "Origin, X-Requested-With, Content-Type, Accept, Authorization, X-Matrix-User-ID"
        Access-Control-Expose-Headers "Content-Length, X-Matrix-User-ID"
    }

    # Matrix discovery
    handle /.well-known/matrix/client {
        header Content-Type application/json
        respond `{"m.homeserver":{"base_url":"https://sky0cloud.dpdns.org"}}`
    }

    handle /.well-known/matrix/server {
        header Content-Type application/json
        respond `{"m.server":"sky0cloud.dpdns.org:443"}`
    }

    # Matrix API paths
    @matrix path /_matrix/* /_synapse/* /_admin/*

    reverse_proxy @matrix sky0cloud-synapse:8008 {
        header_up Host {host}
        header_up X-Real-IP {remote_host}
        header_up X-Forwarded-For {remote_host}

        # Strip duplicate CORS headers
        header_down -Access-Control-Allow-Origin
        header_down -Access-Control-Allow-Methods
        header_down -Access-Control-Allow-Headers
    }

    # Element web client (host port 8090)
    reverse_proxy localhost:8090
}
```

✅ Notes:

* Element Web now listens on **8090** to avoid conflicts.
* All Synapse headers are cleaned so CORS issues won’t occur.

---

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

---

## 4) The "Zero-Downtime" Postgres Migration

Instead of manual snapshots, use the **Docker Stream** method. It’s faster and avoids file permission issues.

### Step A: Prepare the Postgres Config

1. Create `homeserver-postgres.yaml` with your `psycopg2` settings.
2. Ensure the `database:` section points to:

```yaml
host: sky0cloud-postgres
user: synapse_user
password: "Sky@Cloud0Tho"
dbname: synapse
port: 5432
```

### Step B: Run the Port Script

Stop Synapse so the database is quiet, then migrate:

```bash
docker compose stop sky0cloud-synapse

docker compose run --rm sky0cloud-synapse python3 -m synapse.app.port_db \
  --sqlite-database /data/homeserver.db \
  --postgres-config /data/homeserver-postgres.yaml
```

### Step C: The Cutover

```bash
mv synapse/homeserver.yaml synapse/homeserver.yaml.bak
cp synapse/homeserver-postgres.yaml synapse/homeserver.yaml
docker compose up -d sky0cloud-synapse
```

---

## 5) Validation Checklist (The "Success" Test)

Run these commands. If any return a `400` or `502`, the tunnel/proxy is misconfigured.

| Test           | Command                                                             | Expected Result                        |
| -------------- | ------------------------------------------------------------------- | -------------------------------------- |
| **Federation** | `curl https://sky0cloud.dpdns.org/_matrix/client/versions`          | JSON with `versions` field             |
| **Admin API**  | `curl https://sky0cloud.dpdns.org/_synapse/admin/v1/server_version` | JSON with `server_version`             |
| **CORS Check** | `curl -I https://sky0cloud.dpdns.org/_matrix/client/versions`       | Only ONE `Access-Control-Allow-Origin` |

---

## 6) Security Hardening

After your first login, perform these three actions immediately:

1. **Disable Public Registration:** Set `enable_registration: false` in `homeserver.yaml`.
2. **Rotate Secrets:** Change `macaroon_secret_key` and `registration_shared_secret` if previously committed to GitHub.
3. **Cloudflare Access:** Protect `matrix.sky0cloud.dpdns.org` (Admin UI) with **OTP/Email** or Cloudflare Access.

---

**Pro Tip:**
If you see `psycopg2.OperationalError: FATAL: password authentication failed`, check `docker-compose.yml` to ensure the Postgres password matches exactly. Special characters matter.

---
