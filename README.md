# Sky0Cloud Matrix/Synapse Deployment Guide

This guide overhauls the deployment for:
- `https://sky0cloud.dpdns.org` (Element + Matrix API)
- `https://matrix.sky0cloud.dpdns.org` (Synapse Admin UI)

It fixes routing/CORS issues, email verification link behavior, public room publication rules, and migrates from SQLite to PostgreSQL.

## 1) Deploy the optimized stack

```bash
docker compose pull
docker compose up -d
```

Check status:

```bash
docker compose ps
docker compose logs -f caddy
docker compose logs -f synapse
docker compose logs -f postgres
```

## 2) Caddy routing model (clean CORS + React-admin compatibility)

The `Caddyfile` now:
- serves `sky0cloud.dpdns.org` with Element on `/`
- proxies `/_matrix/*` and `/_synapse/*` to Synapse
- serves `matrix.sky0cloud.dpdns.org` from `synapse-admin`
- strips upstream CORS headers and sets a single CORS policy to avoid duplicated `Access-Control-Allow-Origin`

Apply/reload:

```bash
docker compose exec caddy caddy validate --config /etc/caddy/Caddyfile
docker compose exec caddy caddy reload --config /etc/caddy/Caddyfile
```

## 3) Synapse email/password-reset/verification settings

In `synapse/homeserver.yaml`, ensure all of these are correct:

- `public_baseurl: "https://sky0cloud.dpdns.org/"`
- listener has `x_forwarded: true`
- SMTP `email:` block configured for Mailgun
- `email.client_base_url: "https://sky0cloud.dpdns.org"`
- `password_config.enabled: true`
- `account_threepid_delegates: {}`

Then restart Synapse:

```bash
docker compose restart synapse
docker compose logs -f synapse
```

## 4) Fix public room listing permission errors (403)

`room_list_publication_rules` are included to allow local users:

```yaml
room_list_publication_rules:
  - user_id: "@*:sky0cloud.dpdns.org"
    action: allow
  - action: deny
```

## 5) SQLite -> PostgreSQL migration (safe sequence)

> IMPORTANT: do not run `VACUUM` on SQLite during migration.

### Step A - create a PostgreSQL migration config

```bash
cp synapse/homeserver.yaml synapse/homeserver-postgres.yaml
```

Ensure `homeserver-postgres.yaml` has PostgreSQL `database:` settings.

### Step B - initial snapshot migration while service is online

```bash
docker compose stop synapse
cp synapse/data/homeserver.db synapse/data/homeserver.db.snapshot
docker compose start synapse

docker compose run --rm \
  -v "$PWD/synapse:/data" \
  synapse \
  synapse_port_db \
  --sqlite-database /data/data/homeserver.db.snapshot \
  --postgres-config /data/homeserver-postgres.yaml
```

### Step C - final delta migration (short downtime)

```bash
docker compose stop synapse

docker compose run --rm \
  -v "$PWD/synapse:/data" \
  synapse \
  synapse_port_db \
  --sqlite-database /data/data/homeserver.db \
  --postgres-config /data/homeserver-postgres.yaml
```

### Step D - cut over to PostgreSQL permanently

```bash
cp synapse/homeserver-postgres.yaml synapse/homeserver.yaml
docker compose up -d synapse
```

### Step E - verify DB backend

```bash
docker compose logs -f synapse | rg -i "postgres|psycopg2|database"
```

## 6) Validation checklist

```bash
curl -i https://sky0cloud.dpdns.org/_matrix/client/versions
curl -i https://sky0cloud.dpdns.org/_synapse/admin/v1/server_version
curl -i https://sky0cloud.dpdns.org/_matrix/client/v3/account/password/email/requestToken
curl -i https://matrix.sky0cloud.dpdns.org
```

Expected:
- no duplicate CORS headers
- admin panel loads without route loops
- password reset token endpoint is no longer `404 M_UNRECOGNIZED`

## 7) Security hardening (must-do)

Rotate secrets in `synapse/homeserver.yaml`:
- `macaroon_secret_key`
- `registration_shared_secret`
- `form_secret`

Then force token/session invalidation:

```bash
docker compose exec synapse register_new_matrix_user --help >/dev/null
# rotate secrets first, then restart
docker compose restart synapse
```

Recommended extras:
- Keep admin registration closed (`enable_registration: false`) after bootstrap.
- Use long random passwords for SMTP/DB and store with Docker secrets or env files not committed to git.
- Restrict admin panel access at Cloudflare Zero Trust (Access policy) in addition to Synapse auth.
- Keep regular encrypted backups of `synapse/data` and `postgres/data`.
