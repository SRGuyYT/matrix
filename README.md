---

# Sky0Cloud Matrix Stack (Continuwuity Edition)

This repository contains a **full Matrix homeserver stack** running **Continuwuity**, **Element Web**, and **Coturn** with **automatic Postgres backups**, designed to work behind **Cloudflare tunnels**.

---

## Directory Layout

```
matrix/
├── backup/                  # Daily Postgres backups
├── Caddyfile                # Reverse proxy config
├── data/
│   ├── caddy/
│   ├── coturn/
│   │   └── turnserver.conf
│   ├── element/
│   │   ├── config.sky0cloud.dpdns.org.json
│   │   └── icon.png
│   ├── continuwuity/        # Continuwuity database
│   └── postgres/
├── docker-compose.yml
└── README.md
```

> ⚠️ Make sure `data/postgres` and `data/continuwuity` are **empty and writable** by Docker:
>
> ```bash
> sudo rm -rf ./data/postgres/* ./data/continuwuity/*
> sudo chown -R 999:999 ./data/postgres
> ```

---
