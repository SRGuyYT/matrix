# Sky0Cloud Matrix Stack (Cloudflare Tunnel + Caddy HTTP Reverse Proxy)

This repository deploys a Matrix stack for `sky0cloud.dpdns.org` tuned for low write-amplification hosts (8 GB RAM + HDD) and Cloudflare-terminated TLS.

## Services

- `synapse` (Matrix homeserver) on internal `:8008`
- `element-web` (Element client) on internal `:80`
- `synapse-admin` (admin GUI) on host/internal `:8000`
- `caddy` (reverse proxy, **port 80 only**, no auto HTTPS)
- `cloudflared` (Cloudflare tunnel ingress)

## Key behavior

- Public chat URL: `https://sky0cloud.dpdns.org`
- Public admin URL: `https://matrix.sky0cloud.dpdns.org`
- Caddy listens on `:80` only and does not create certs.
- Cloudflare handles TLS externally.

## Why this fixes prior issues

- Password reset endpoint `/_matrix/client/v3/account/password/email/requestToken` requires homeserver support + SMTP; Synapse provides this.
- Element welcome page now uses a minimal valid `config.json` with correct homeserver URLs.
- Compose logging is rotated (`max-size`, `max-file`) to reduce HDD pressure.

## Cloudflare tunnel

Edit `cloudflared/config.yml` and set:

- `tunnel` ID
- `credentials-file`

Ingress routes include both hostnames:

- `sky0cloud.dpdns.org -> http://caddy:80`
- `matrix.sky0cloud.dpdns.org -> http://caddy:80`

## SMTP/password reset

Set real SMTP credentials in `synapse/homeserver.yaml`:

- `email.smtp_pass`
- secrets (`registration_shared_secret`, `macaroon_secret_key`, `form_secret`)

Then restart `synapse`.

## Start and validate

```bash
docker compose up -d

docker compose ps
docker compose logs -f synapse
curl -s http://localhost/_matrix/client/versions
curl -s http://localhost/_matrix/client/v3/account/password/email/requestToken
```
