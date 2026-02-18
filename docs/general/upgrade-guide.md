# Upgrade Guide

This guide highlights important changes when upgrading to **DSMR Reader Docker v6**.

> ✅ **New installations usually require no changes.**
> This section is mainly relevant when upgrading from DSMR Reader Docker v5.

---

## Summary of Breaking Changes

DSMR Reader Docker v6 introduces:

- A new `CONTAINER_` prefix for Docker specific variables
- New default UID and GID (`1000:1000`)
- Updated timezone handling

⚠️ Review this guide carefully if you are reusing existing volumes.

---

## Variables changes

From **DSMR Reader Docker v6**, all **Docker-specific** environment variables use the `CONTAINER_` prefix.
This clearly separates container behavior from DSMR Reader’s own settings.

⚠️ When migrating to v6, review your environment variables carefully.
DSMR Reader variables themselves may also have changed between major versions.

### DSMR Reader Docker-specific variables (`CONTAINER_*`)

- `CONTAINER_RUN_MODE`
- `CONTAINER_ENABLE_DEBUG`
- `CONTAINER_ENABLE_NGINX_ACCESS_LOGS`
- `CONTAINER_ENABLE_NGINX_SSL`
- `CONTAINER_ENABLE_HTTP_AUTH`
- `CONTAINER_ENABLE_CLIENTCERT_AUTH`
- `CONTAINER_ENABLE_IFRAME`
- `CONTAINER_ENABLE_VACUUM_DB_AT_STARTUP`

---

## UID and GID changes

In DSMR Reader Docker v6 the default user and group IDs changed to:

- **v6 default:** `1000:1000`
- **v5 default:** `803:<group>`

This only affects **filesystem permissions on existing volumes**.
New installations are not impacted.

### How to fix

If you experience permission problems after upgrading, explicitly set the IDs to match your existing data.

You can use either of the following (both are supported):

**Option 1**
```
PUID=803
PGID=1000
```

**Option 2**
```
DUID=803
DGID=1000
```

💡 Tip: Only change these if you already have existing volumes created with the old UID/GID.
New installations should normally keep the defaults.

---

## Timezone configuration changes

### ❌ Do not use these volume mounts anymore

Remove these if present:

- `/etc/localtime:/etc/localtime`
- `/etc/timezone:/etc/timezone`

### ❌ Remove these environment variables

Remove if defined:

- `TZ=Europe/Amsterdam`
- `PG_TZ=Europe/Amsterdam`

### ✅ Recommended approach

Use Django’s timezone setting instead:

```
DJANGO_TIME_ZONE=Europe/Amsterdam
```

---

## Upgrade Procedure

Typical safe upgrade flow:

```bash
# 1. Backup database (recommended)
docker exec dsmrdb pg_dump -U dsmrreader dsmrreader > backup.sql

# 2. Pull new image
docker compose pull

# 3. Recreate containers
docker compose up -d
```

Then verify:

```bash
docker compose ps
docker compose logs dsmr
```

---

## After Upgrading

Verify the following:

- Containers start without errors
- Web interface loads
- Telegrams are being processed
- No permission errors in logs

Health check:

```bash
curl http://localhost/healthcheck
```

Should return HTTP 200.

