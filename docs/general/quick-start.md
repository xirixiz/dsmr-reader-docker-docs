# Quick Start

Create `docker-compose.yaml`:

```yaml
volumes:
  dsmrdb_data:

services:
  dsmrdb:
    image: postgres:17-alpine
    restart: always
    volumes:
      - dsmrdb_data:/var/lib/postgresql/data
    environment:
      POSTGRES_USER: dsmrreader
      POSTGRES_PASSWORD: dsmrreader
      POSTGRES_DB: dsmrreader

  dsmr:
    image: xirixiz/dsmr-reader-docker:latest
    restart: always
    depends_on:
      - dsmrdb
    ports:
      - "80:80" # you may change this, e.g. "7777:80"
    devices:
      - /dev/ttyUSB0:/dev/ttyUSB0
    environment:
      CONTAINER_RUN_MODE: standalone
      DJANGO_DATABASE_HOST: dsmrdb
      DJANGO_DATABASE_NAME: dsmrreader
      DJANGO_DATABASE_USER: dsmrreader
      DJANGO_DATABASE_PASSWORD: dsmrreader
      DJANGO_TIME_ZONE: Europe/Amsterdam
      DJANGO_SECRET_KEY: change-me-to-random-string
      DSMRREADER_ADMIN_USER: admin
      DSMRREADER_ADMIN_PASSWORD: admin
```

Start it:

```bash
docker compose up -d
```

Access the web interface at `http://localhost`
(login: `admin` / `admin`)

---

## Common Variant: Network Smart Meter (Pull)

If your smart meter is exposed over the network (for example via ser2net or a P1 TCP gateway), you can let DSMR Reader **pull** the data.

Add these environment variables to the `dsmr` service:

```yaml
environment:
  CONTAINER_RUN_MODE: standalone
  DSMRREADER_REMOTE_DATALOGGER_INPUT_METHOD: ipv4
  DSMRREADER_REMOTE_DATALOGGER_NETWORK_HOST: 192.168.1.100
  DSMRREADER_REMOTE_DATALOGGER_NETWORK_PORT: 23
```

✅ Still one container
✅ No remote_datalogger mode required
✅ No API configuration required

---

## Key Features

- 🚀 **s6-overlay v3** – Robust process supervision
- 🏗️ **Multi-arch** – amd64, arm32v7, arm64v8
- 🔄 **Flexible modes** – standalone, server_remote_datalogger, remote_datalogger
- 📊 **PostgreSQL** – Reliable data storage
- 🔌 **Serial or network** – Connect via USB or TCP/IP
