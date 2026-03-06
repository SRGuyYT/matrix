# Sky0Cloud: Matrix Stack Behind Cloudflare Tunnel

This repository provides a clean, production-ready Matrix deployment for `sky0cloud.dpdns.org` using Docker Compose.

## Architecture

```text
Internet
  ↓
Cloudflare Tunnel
  ↓
Caddy (HTTP only, port 80)
  ↓
Continuwuity Matrix Homeserver
  ↓
Element Web Client
```

## Project structure

```text
Sky0Cloud/
├── docker-compose.yml
├── Caddyfile
├── README.md
├── continuwuity/
│   └── data/
├── element/
│   └── config.json
└── cloudflared/
    └── config.yml
```

## Services

- `continuwuity` - Matrix homeserver (port `8008` internally).
- `element-web` - Matrix web client.
- `caddy` - HTTP-only reverse proxy and Matrix well-known endpoint handler.
- `cloudflared` - Cloudflare Tunnel ingress connector.

All services are attached to the internal Docker bridge network: `sky0cloud-network`.

## Prerequisites

- Docker Engine + Docker Compose plugin installed.
- Cloudflare account and Zero Trust Tunnel configured.
- Domain `sky0cloud.dpdns.org` delegated/managed in Cloudflare.

## 1) Cloudflare Tunnel setup

### Option A: Tunnel token (recommended)

1. In Cloudflare Zero Trust, create a tunnel.
2. Add a public hostname:
   - `sky0cloud.dpdns.org` -> HTTP -> `caddy:80`
3. Copy the tunnel token.
4. Add it to `docker-compose.yml` for `cloudflared` by replacing command with:

```yaml
command: tunnel --no-autoupdate run --token ${CLOUDFLARE_TUNNEL_TOKEN}
```

5. Create `.env` next to `docker-compose.yml`:

```env
CLOUDFLARE_TUNNEL_TOKEN=your_token_here
```

### Option B: Named tunnel credentials file

1. Place tunnel credentials JSON in `./cloudflared/`.
2. Extend `cloudflared/config.yml` with:

```yaml
tunnel: <YOUR_TUNNEL_UUID>
credentials-file: /etc/cloudflared/<YOUR_TUNNEL_UUID>.json
```

## 2) Start the stack

From the repository root:

```bash
docker compose pull
docker compose up -d
```

Check status:

```bash
docker compose ps
docker compose logs -f caddy
docker compose logs -f continuwuity
docker compose logs -f cloudflared
```

## 3) Routing behavior

Caddy runs **HTTP only** (no TLS, no certificate management).

- `/.well-known/matrix/client` -> JSON response for Matrix client discovery.
- `/.well-known/matrix/server` -> JSON response for Matrix server discovery.
- `/.well-known/matrix/*` -> proxied to `continuwuity:8008`.
- `/_matrix/*` -> proxied to `continuwuity:8008`.
- `/` -> proxied to `element-web:80`.

Cloudflare provides external HTTPS and forwards traffic through the tunnel to `http://caddy:80`.

## 4) Federation and Matrix validation

### Verify well-known endpoints

```bash
curl -s https://sky0cloud.dpdns.org/.well-known/matrix/client | jq
curl -s https://sky0cloud.dpdns.org/.well-known/matrix/server | jq
```

Expected values:
- `m.homeserver.base_url` = `https://sky0cloud.dpdns.org`
- `m.server` = `sky0cloud.dpdns.org:443`

### Verify Matrix client API reachability

```bash
curl -s https://sky0cloud.dpdns.org/_matrix/client/versions | jq
```

### Federation test

Use Matrix federation tester:

- https://federationtester.matrix.org/#sky0cloud.dpdns.org

You should see successful federation checks for SRV/delegation and Matrix endpoints.

## 5) Functional checklist

After deployment, verify:

- Element loads at `https://sky0cloud.dpdns.org`
- User registration/login works
- Room creation works
- Media upload and download work (`/_matrix/media/*`)
- Federation joins/public rooms work across external servers

## 6) Persistent data

- Matrix homeserver data is persisted under `./continuwuity/data`.
- Caddy runtime data/config are persisted in Docker managed volumes (`caddy_data`, `caddy_config`).

## 7) Troubleshooting

### Tunnel connected but site unavailable

- Check tunnel status in Cloudflare Zero Trust dashboard.
- Confirm `cloudflared` can resolve `caddy` on Docker network.
- Verify `cloudflared/config.yml` ingress points to `http://caddy:80`.

### Element loads but login fails

- Confirm `element/config.json` base URL matches `https://sky0cloud.dpdns.org`.
- Confirm Caddy routes `/_matrix/*` to `continuwuity:8008`.
- Check homeserver logs: `docker compose logs -f continuwuity`.

### Federation fails

- Confirm `/.well-known/matrix/server` returns `{"m.server":"sky0cloud.dpdns.org:443"}`.
- Ensure Cloudflare Tunnel hostname exactly matches `sky0cloud.dpdns.org`.
- Run federation tester and review failing checks.

### Media uploads/downloads fail

- Confirm `MAX_REQUEST_SIZE=524288000` is applied.
- Ensure reverse proxy forwards `/_matrix/media/*` (covered by `/_matrix/*`).
- Check container disk space and write permissions in `./continuwuity/data`.

## 8) Update and maintenance

```bash
docker compose pull
docker compose up -d
```

To restart a single service:

```bash
docker compose restart continuwuity
```

To stop the stack:

```bash
docker compose down
```
