# Advanced

Advanced configuration options for production deployments.

---

## SSL/TLS Configuration

Enable HTTPS for secure access to DSMR Reader.

### Generate Certificates

**Self-signed (testing only):**
```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout privkey.pem -out fullchain.pem \
  -subj "/CN=dsmr.local"
```

**Let's Encrypt (production):**
```bash
certbot certonly --standalone -d dsmr.example.com
```

---

### Configure Container

> The container expects certificates at:
> - `/etc/ssl/private/fullchain.pem`
> - `/etc/ssl/private/privkey.pem`

```yaml
services:
  dsmr:
    image: xirixiz/dsmr-reader-docker:latest
    ports:
      - "443:443"
      - "80:80" # optional
    volumes:
      - ./certs/fullchain.pem:/etc/ssl/private/fullchain.pem:ro
      - ./certs/privkey.pem:/etc/ssl/private/privkey.pem:ro
    environment:
      CONTAINER_ENABLE_NGINX_SSL: "true"
```

---

## HTTP Basic Authentication

Protect the web interface with username/password.

> The container automatically generates the htpasswd file.

```yaml
environment:
  CONTAINER_ENABLE_HTTP_AUTH: "true"
  HTTP_AUTH_USERNAME: "username"
  HTTP_AUTH_PASSWORD: "change-me"
```

---

## Client Certificate Authentication

Mutual TLS authentication using client certificates.

### Requirements

Mount your CA certificate to:

- `/etc/nginx/client_cert/cacert.pem`
Optional CRL:

- `/etc/nginx/client_cert/ca.crl`

---

### Configure Container

```yaml
services:
  dsmr:
    image: xirixiz/dsmr-reader-docker:latest
    ports:
      - "443:443"
    volumes:
      - ./certs/fullchain.pem:/etc/ssl/private/fullchain.pem:ro
      - ./certs/privkey.pem:/etc/ssl/private/privkey.pem:ro
      - ./client-ca/cacert.pem:/etc/nginx/client_cert/cacert.pem:ro
      # optional:
      # - ./client-ca/ca.crl:/etc/nginx/client_cert/ca.crl:ro
    environment:
      CONTAINER_ENABLE_NGINX_SSL: "true"
      CONTAINER_ENABLE_CLIENTCERT_AUTH: "true"
```

**Note:** Users must import their client certificate into their browser.

---

## Network Smart Meters (Pull)

Read smart meters via TCP/IP instead of USB serial.

✅ Works in `standalone`
✅ Works in `server_remote_datalogger`
❌ Does NOT require `remote_datalogger` mode
❌ Does NOT require API keys

---

### Configuration

```yaml
environment:
  CONTAINER_RUN_MODE: standalone
  DSMRREADER_REMOTE_DATALOGGER_INPUT_METHOD: ipv4
  DSMRREADER_REMOTE_DATALOGGER_NETWORK_HOST: 192.168.1.100
  DSMRREADER_REMOTE_DATALOGGER_NETWORK_PORT: 23
```

---

### Common Devices

- Network serial adapters
- ser2net
- P1 TCP gateways

---

## Iframe Embedding

Allow embedding DSMR Reader in dashboards (Home Assistant, Grafana, etc).

```yaml
environment:
  CONTAINER_ENABLE_IFRAME: "true"
```

⚠️ Only enable on trusted networks.

---

## Database Maintenance

### Automatic Vacuum on Startup

Enable database optimization on container startup:

```yaml
environment:
  CONTAINER_ENABLE_VACUUM_DB_AT_STARTUP: "true"
```

Note: increases startup time but improves performance for large databases.

---

### Manual Vacuum

```bash
docker exec dsmr /app/cleandb.sh
```

Verbose:

```bash
docker exec dsmr /app/cleandb.sh -v
```

---

## Reverse Proxy Setup

### Important Django Settings

When running behind a reverse proxy, you may need:

```yaml
environment:
  DJANGO_ALLOWED_HOSTS: "*"
  # Only set if you use a fixed external URL
  # DJANGO_CSRF_TRUSTED_ORIGINS: "https://dsmr.example.com"
```

---

### Nginx Example

```nginx
server {
    listen 443 ssl http2;
    server_name dsmr.example.com;

    ssl_certificate /etc/nginx/ssl/cert.pem;
    ssl_certificate_key /etc/nginx/ssl/key.pem;

    location / {
        proxy_pass http://localhost:80;
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

---

### Traefik Example

```yaml
services:
  dsmr:
    image: xirixiz/dsmr-reader-docker:latest
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.dsmr.rule=Host(`dsmr.example.com`)"
      - "traefik.http.routers.dsmr.entrypoints=websecure"
      - "traefik.http.routers.dsmr.tls.certresolver=letsencrypt"
```

---

## Debug Mode

Enable verbose container initialization logging:

```yaml
environment:
  CONTAINER_ENABLE_DEBUG: "true"
```

For DSMR Reader application debug, see upstream documentation.

---

## Custom Plugins

Load custom DSMR Reader plugins:

```yaml
volumes:
  - ./plugins/my_plugin.py:/app/dsmr_plugins/modules/my_plugin.py:ro
environment:
  DSMRREADER_PLUGINS: dsmr_plugins.modules.my_plugin
```

Multiple plugins:

```yaml
environment:
  DSMRREADER_PLUGINS: dsmr_plugins.modules.plugin1,dsmr_plugins.modules.plugin2
```

---

## Backup Strategy

### Database Backups

**Method 1:**

```bash
docker exec dsmr sh -c 'PGPASSWORD=$DJANGO_DATABASE_PASSWORD \
  pg_dump -h $DJANGO_DATABASE_HOST -U $DJANGO_DATABASE_USER \
  $DJANGO_DATABASE_NAME' > backup.sql
```

**Method 2:**

```bash
docker exec dsmrdb pg_dump -U dsmrreader dsmrreader > backup.sql
```

**Method 3:**

```bash
docker run --rm \
  -v dsmrdb_data:/volume \
  -v $(pwd):/backup \
  alpine tar czf /backup/dsmrdb.tar.gz -C /volume ./
```

---

## Automated Backups

Example cron job:

```bash
#!/bin/bash
BACKUP_DIR=/backups/dsmr
DATE=$(date +%Y%m%d_%H%M%S)
docker exec dsmrdb pg_dump -U dsmrreader dsmrreader > ${BACKUP_DIR}/dsmr_${DATE}.sql
find ${BACKUP_DIR} -name "dsmr_*.sql" -mtime +7 -delete
```

---

## Restore

```bash
docker compose stop dsmr
cat backup.sql | docker exec -i dsmrdb psql -U dsmrreader -d dsmrreader
docker compose start dsmr
```
