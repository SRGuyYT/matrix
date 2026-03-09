* Run on `sky0cloud.dpdns.org`
* Run Element Web and Synapse properly behind Cloudflare tunnels
* Use TURN server for WebRTC calls (`coturn.sky0cloud.dpdns.org:3478`)
* Include automatic daily Postgres backups
* Avoid volume/mount issues that were breaking Postgres

---

## 1️⃣ Directory Layout

```
matrix/
├── backup/
├── Caddyfile
├── data/
│   ├── caddy/
│   ├── coturn/
│   │   └── turnserver.conf
│   ├── element/
│   │   ├── config.json
│   │   └── icon.png
│   ├── postgres/
│   ├── redis/
│   └── synapse/
│       └── homeserver.yaml
├── docker-compose.yml
└── README.md
```

Make sure `data/postgres` is empty **and writable** by your Docker user. Example:

```bash
sudo rm -rf ./data/postgres/*
sudo chown -R 999:999 ./data/postgres
```

---

## 2️⃣ Docker Compose

```yaml
version: "3.9"
services:
  postgres:
    image: postgres:16-alpine
    container_name: sky0cloud-postgres
    restart: unless-stopped
    environment:
      POSTGRES_USER: synapse_user
      POSTGRES_PASSWORD: Sky0CloudStrongDBPass
      POSTGRES_DB: synapse
    volumes:
      - ./data/postgres:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U synapse_user -d synapse"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    container_name: sky0cloud-redis
    restart: unless-stopped
    command: ["redis-server","--appendonly","yes"]
    volumes:
      - ./data/redis:/data

  synapse:
    image: matrixdotorg/synapse:latest
    container_name: sky0cloud-synapse
    restart: unless-stopped
    environment:
      SYNAPSE_SERVER_NAME: sky0cloud.dpdns.org
      SYNAPSE_CONFIG_PATH: /data/homeserver.yaml
    volumes:
      - ./data/synapse:/data
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_started

  element:
    image: vectorim/element-web:latest
    container_name: sky0cloud-element
    restart: unless-stopped
    volumes:
      - ./data/element:/app/config

  synapse-admin:
    image: ghcr.io/etkecc/synapse-admin:latest
    container_name: sky0cloud-synapse-admin
    restart: unless-stopped
    environment:
      SYNAPSE_SERVER: http://sky0cloud-synapse:8008
    depends_on:
      - synapse

  coturn:
    image: coturn/coturn:latest
    container_name: sky0cloud-turn
    restart: unless-stopped
    network_mode: host
    volumes:
      - ./data/coturn/turnserver.conf:/etc/coturn/turnserver.conf
    command: ["-c", "/etc/coturn/turnserver.conf"]

  postgres-backup:
    image: prodrigestivill/postgres-backup-local
    container_name: sky0cloud-postgres-backup
    restart: unless-stopped
    environment:
      POSTGRES_HOST: postgres
      POSTGRES_DB: synapse
      POSTGRES_USER: synapse_user
      POSTGRES_PASSWORD: Sky0CloudStrongDBPass
      SCHEDULE: "@daily"
    volumes:
      - ./backup:/backups
    depends_on:
      - postgres

  caddy:
    image: caddy:2-alpine
    container_name: sky0cloud-caddy
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile
      - ./data/caddy:/data
      - ./data/caddy_config:/config
    depends_on:
      - synapse
      - element
```

---

## 3️⃣ Homeserver (`homeserver.yaml`)

```yaml
server_name: "sky0cloud.dpdns.org"
public_baseurl: "https://sky0cloud.dpdns.org/"

pid_file: /data/homeserver.pid
signing_key_path: "/data/sky0cloud.dpdns.org.signing.key"

report_stats: true
suppress_key_server_warning: true

listeners:
  - port: 8008
    tls: false
    type: http
    x_forwarded: true
    bind_addresses: ["0.0.0.0"]
    resources:
      - names: [client, federation]
        compress: true

database:
  name: psycopg2
  args:
    user: synapse_user
    password: Sky0CloudStrongDBPass
    dbname: synapse
    host: postgres
    port: 5432
    cp_min: 5
    cp_max: 15

redis:
  enabled: true
  host: redis
  port: 6379

media_store_path: /data/media_store
uploads_path: /data/uploads
max_upload_size: 100M
max_image_pixels: 32M

enable_registration: true
registration_requires_token: true
registration_shared_secret: Sky0CloudInviteSecret
allow_guest_access: false

trusted_proxies:
  - "127.0.0.1"
  - "172.16.0.0/12"
  - "10.0.0.0/8"

trusted_key_servers:
  - server_name: "matrix.org"

auto_join_rooms:
  - "#general:sky0cloud.dpdns.org"
  - "#announcements:sky0cloud.dpdns.org"

server_notices:
  system_mxid_localpart: "system"
  system_mxid_display_name: "Sky0Cloud Account Notice Team"
  room_name: "[Official] Account Notice"

turn_uris:
  - "turn:coturn.sky0cloud.dpdns.org:3478?transport=udp"
  - "turn:coturn.sky0cloud.dpdns.org:3478?transport=tcp"

turn_shared_secret: Sky0CloudTurnSecret
turn_user_lifetime: 86400000
turn_allow_guests: false

federation:
  client_timeout: 60s
  max_long_retry_delay: 60s

rc_message:
  per_second: 0.5
  burst_count: 30

rc_login:
  address:
    per_second: 0.17
    burst_count: 3

rc_registration:
  per_second: 0.17
  burst_count: 3

caches:
  global_factor: 1.5

event_cache_size: 200K

allow_public_rooms_over_federation: true
allow_public_rooms_without_auth: true

room_list_publication_rules:
  - user_id: "@system:sky0cloud.dpdns.org"
    alias: "*"
    action: "allow"
```

---

## 4️⃣ Element Web (`config.json`)

```json
{
  "brand": "Sky0Cloud",
  "default_server_name": "sky0cloud.dpdns.org",
  "default_server_config": {
    "m.homeserver": {
      "base_url": "https://sky0cloud.dpdns.org",
      "server_name": "sky0cloud.dpdns.org"
    },
    "m.identity_server": {
      "base_url": "https://vector.im"
    }
  },
  "room_directory": {
    "servers": [
      "sky0cloud.dpdns.org",
      "matrix.org"
    ]
  },
  "features": {
    "threadsActivityCentre": true,
    "feature_video_rooms": true,
    "feature_group_calls": true
  },
  "default_theme": "dark",
  "branding": {
    "authHeaderLogoUrl": "icon.png"
  },
  "webrtc": {
    "preferredCodec": "VP8",
    "turnServers": [
      {
        "urls": [
          "turn:coturn.sky0cloud.dpdns.org:3478?transport=udp",
          "turn:coturn.sky0cloud.dpdns.org:3478?transport=tcp"
        ],
        "username": "sky0cloud_turn_user",
        "credential": "Sky0CloudTurnSecret"
      }
    ]
  }
}
```

---

## 5️⃣ Coturn (`turnserver.conf`)

```conf
realm=sky0cloud.dpdns.org
use-auth-secret
static-auth-secret=Sky0CloudTurnSecret
fingerprint

listening-port=3478
min-port=49152
max-port=65535

external-ip=108.82.191.18
```

---

✅ **Next steps / tips**:

1. Make sure `data/postgres` is **empty and writable** (`chown -R 999:999 data/postgres`).
2. Start the stack:

```bash
docker compose down -v
docker compose up -d
docker compose logs -f
```

3. Check `sky0cloud.dpdns.org` — Element should load and Synapse API calls should succeed.
4. TURN should now work for WebRTC calls with the same secret as in Synapse & Element.

---
