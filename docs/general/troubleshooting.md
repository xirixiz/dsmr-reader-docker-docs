# Troubleshooting

Common issues and solutions for DSMR Reader Docker.

---

## Quick Diagnostics

```bash
# Check container status
docker compose ps

# View container logs
docker compose logs dsmr
docker compose logs dsmrdb

# Check s6 overlay services
docker exec dsmr s6-rc -a list

# Test Django database connectivity
docker exec dsmr /opt/venv/bin/python /app/manage.py check --database default
```

---

## Container Won't Start

### Check logs

```bash
docker compose logs dsmr
```

### Enable debug mode

```yaml
environment:
  CONTAINER_ENABLE_DEBUG: "true"
```

### Check service status

```bash
docker exec dsmr s6-rc -a list
```

### Common causes

1. Database not ready: container starts before Postgres is accepting connections
   Solution: ensure Postgres is in the same compose file and DSMR uses the service name as host, for example `DJANGO_DATABASE_HOST: dsmrdb`

2. Missing environment variables: required variables not set
   Solution: verify your configuration reference

3. Port conflict: host port already in use
   Solution: change port mapping, for example `7777:80`

---

## Database Connection Issues

### Symptoms

* Web interface shows database errors
* Connection refused in logs
* FATAL: database does not exist

### Solutions

Check database is running:

```bash
docker compose ps dsmrdb
docker compose logs dsmrdb
```

Verify credentials match:

```text
POSTGRES_USER matches DJANGO_DATABASE_USER
POSTGRES_PASSWORD matches DJANGO_DATABASE_PASSWORD
POSTGRES_DB matches DJANGO_DATABASE_NAME
```

Test connection from the DSMR container:

```bash
docker exec -e PGPASSWORD="$DJANGO_DATABASE_PASSWORD" dsmr \
  psql -h "$DJANGO_DATABASE_HOST" -U "$DJANGO_DATABASE_USER" -d "$DJANGO_DATABASE_NAME"
```

Check database host:

```yaml
# For Docker Compose on the same host, use the service name
DJANGO_DATABASE_HOST: dsmrdb

# Do not use localhost or 127.0.0.1 for the database host
```

---

## Web Interface Not Accessible

### Check nginx and gunicorn are running

```bash
docker exec dsmr ps aux | grep -E "nginx|gunicorn"
```

### Check port mapping

```bash
docker compose ps
```

If port 80 is in use, map a different host port:

```yaml
ports:
  - "7777:80"
```

### Nginx access logs and error logs

This image writes nginx logs to container stdout and stderr. You can view them with:

```bash
docker compose logs -f dsmr
```

If you want access logs enabled:

```yaml
environment:
  CONTAINER_ENABLE_NGINX_ACCESS_LOGS: "true"
```

### Test nginx configuration

```bash
docker exec dsmr nginx -c /etc/nginx/nginx.conf -t
```

---

## Serial Device Not Accessible

### Check device exists on the host

```bash
ls -l /dev/ttyUSB0
```

### Check permissions

```bash
ls -l /dev/ttyUSB0

# Temporary fix
sudo chmod 666 /dev/ttyUSB0
```

### Add your user to dialout group

```bash
sudo usermod -aG dialout "$USER"
# Log out and back in
```

### Verify device is passed to the container

```yaml
devices:
  - /dev/ttyUSB0:/dev/ttyUSB0
```

Check inside the container:

```bash
docker exec dsmr ls -l /dev/ttyUSB0
```

---

## Timestamp Issues

### Timestamps off by one hour

Do not mount host localtime into the container. Use timezone env var:

```yaml
environment:
  DJANGO_TIME_ZONE: Europe/Amsterdam
```

Verify time:

```bash
docker exec dsmr date
docker exec dsmrdb psql -U dsmrreader -d dsmrreader -c "SHOW TIME ZONE;"
```

---

## Data Not Showing or No Readings

### Check datalogger activity

```bash
docker compose logs dsmr | grep -i datalogger
docker compose logs dsmr | grep -i telegram
```

### Verify datalogger configuration in the UI

Web interface then Configuration then Datalogger

Common settings:

* DSMR version: set to your meter version
* Serial port: `/dev/ttyUSB0` or `/dev/dsmr_p1`
* Baud rate: 115200 for DSMR 4 or 5, 9600 for DSMR 2 or 3

### Test serial connection manually

```bash
docker exec -it dsmr bash
cat /dev/ttyUSB0
```

You should see telegram data periodically. Press Ctrl C to stop.

---

## Performance Issues

### Slow web interface

Check database size:

```bash
docker exec dsmrdb psql -U dsmrreader -d dsmrreader \
  -c "SELECT pg_size_pretty(pg_database_size('dsmrreader'));"
```

Enable vacuum on startup:

```yaml
environment:
  CONTAINER_ENABLE_VACUUM_DB_AT_STARTUP: "true"
```

Or run vacuum manually:

```bash
docker exec dsmr /app/cleandb.sh -v
```

### High CPU usage

```bash
docker exec dsmr ps aux
docker stats dsmr dsmrdb
```

---

## SSL Certificate Errors

This image expects certificates at:

* `/etc/ssl/private/fullchain.pem`
* `/etc/ssl/private/privkey.pem`

Verify certificates are mounted:

```bash
docker exec dsmr ls -la /etc/ssl/private/
```

Test nginx configuration:

```bash
docker exec dsmr nginx -c /etc/nginx/nginx.conf -t
```

Check certificate validity:

```bash
docker exec dsmr openssl x509 -in /etc/ssl/private/fullchain.pem -text -noout
```

---

## Remote Datalogger and Remote Input Issues

There are two different scenarios.

### Remote input pull

DSMR Reader reads from a remote TCP source like ser2net or a P1 TCP gateway.

Check connectivity from the DSMR container:

```bash
docker exec dsmr sh -c "nc -zv 192.168.1.100 23"
```

Verify env vars:

```bash
docker exec dsmr env | grep DSMRREADER_REMOTE_DATALOGGER_
```

### Remote datalogger push

A separate forwarder container reads the meter and pushes telegrams to the server via API.

Test server reachability from the forwarder:

```bash
docker exec dsmr-remote curl -fsS http://dsmr-server/healthcheck >/dev/null
```

Check forwarder env vars:

```bash
docker exec dsmr-remote env | grep DSMRREADER_REMOTE_DATALOGGER_API_
```

Check server logs for API activity:

```bash
docker compose logs dsmr-server | grep -i api
```

---

## Container Upgrade Issues

### After upgrade, container will not start

Check breaking changes:

```text
Review GitHub releases for this image and upstream DSMR Reader
```

Check database version:

```bash
docker exec dsmrdb psql -V
```

Restore from backup if needed, see Advanced section.

---

## Platform Specific Issues

### Raspberry Pi

If you see Y2038 or libseccomp related errors, update your host packages. The exact steps depend on your Pi OS version.

---

### Synology NAS

USB serial support depends on DSM version and model. If `/dev/ttyUSB0` is missing, verify the driver situation on the host first.

---

### WSL2

USB devices are not natively accessible. Prefer network smart meter or USB IP forwarding.

---

## Getting More Help

### Enable debug logging

```yaml
environment:
  CONTAINER_ENABLE_DEBUG: "true"
```

### Collect diagnostic information

```bash
docker compose ps
docker compose logs dsmr > dsmr-logs.txt
docker compose logs dsmrdb > db-logs.txt

docker version
docker compose version
uname -a

# Remove sensitive values before sharing
docker compose config > config.yaml
```

### Links

* [GitHub issues for this image](https://github.com/xirixiz/dsmr-reader-docker/issues)
* [GitHub discussions for this image](https://github.com/xirixiz/dsmr-reader-docker/discussions)
* [Upstream DSMR Reader documentation](https://dsmr-reader.readthedocs.io/)
* [Upstream DSMR Reader issues](https://github.com/dsmrreader/dsmr-reader/issues)
* [Upstream DSMR Reader discussions](https://github.com/dsmrreader/dsmr-reader/discussions)
