# Quick Start

Create `docker-compose.yaml`:

```yaml
version: '3.8'

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
      - "80:80"
    devices:
      - /dev/ttyUSB0:/dev/ttyUSB0
    environment:
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
docker-compose up -d
```

Access the web interface at `http://localhost` (login: `admin` / `admin`)

---

## Key Features

- 🚀 **s6-overlay v3** - Robust process supervision
- 🏗️ **Multi-arch** - amd64, arm32v7, arm64v8
- 🔄 **Flexible modes** - Standalone, server_remote_datalogger, remote_datalogger
- 📊 **PostgreSQL** - Reliable data storage
- 🔌 **Serial or network** - Connect via USB or TCP/IP
