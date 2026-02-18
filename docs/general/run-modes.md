# Run Modes

DSMR Reader Docker supports three operational modes to fit different deployment scenarios.

The key difference between modes is how telegrams reach DSMR Reader:

- **Pull model**: DSMR Reader reads the meter itself
- **Push model**: a remote datalogger sends data to DSMR Reader

---

## Mode Comparison

| Mode | Meter Access | Data Flow | Database | Web UI | Typical Use |
|------|-------------|-----------|----------|--------|-------------|
| **standalone** | Local or network | **Pull** | Yes | Yes | All in one setup |
| **server_remote_datalogger** | None | **Receives push** | Yes | Yes | Central server |
| **remote_datalogger** | Local or network | **Push** | No | No | Remote forwarder |

---

## Data Flow Explained

### Pull (recommended for most users)

DSMR Reader reads the meter itself.

Examples:

- USB P1 cable
- ser2net
- TCP P1 gateway
- WiFi P1 dongle

```
[Smart meter] → DSMR Reader
```

Used by:

- standalone

---

### Push (advanced setups)

A separate datalogger reads the meter and sends data via API.

```
[Smart meter] → [Remote datalogger] → [server_remote_datalogger] (DSMR Reader API)
```

Used by:

- remote_datalogger + server_remote_datalogger

---

# Standalone Mode (Default)

**When to use:** single location or simple setups.

Runs the complete DSMR Reader stack:

- database
- web interface
- datalogger

The datalogger can read the meter:

- locally (USB)
- remotely (ser2net or TCP)

⚠️ No API configuration required.

---

## Example: Full standalone stack

```yaml
services:
  dsmrdb:
    image: postgres:17-alpine
    volumes:
      - dsmrdb_data:/var/lib/postgresql/data
    environment:
      POSTGRES_USER: dsmrreader
      POSTGRES_PASSWORD: dsmrreader
      POSTGRES_DB: dsmrreader

  dsmr:
    image: xirixiz/dsmr-reader-docker:latest
    depends_on:
      - dsmrdb
    ports:
      - "80:80"
    devices:
      - /dev/ttyUSB0:/dev/ttyUSB0
    environment:
      CONTAINER_RUN_MODE: standalone
      DJANGO_DATABASE_HOST: dsmrdb
      DJANGO_DATABASE_NAME: dsmrreader
      DJANGO_DATABASE_USER: dsmrreader
      DJANGO_DATABASE_PASSWORD: dsmrreader
      DJANGO_TIME_ZONE: Europe/Amsterdam
      DJANGO_SECRET_KEY: your-secret-key
      DSMRREADER_ADMIN_USER: admin
      DSMRREADER_ADMIN_PASSWORD: admin
```

---

## Example: Remote meter via network (pull)

This still uses **standalone**.

```yaml
environment:
  CONTAINER_RUN_MODE: standalone
  DSMRREADER_REMOTE_DATALOGGER_INPUT_METHOD: ipv4
  DSMRREADER_REMOTE_DATALOGGER_NETWORK_HOST: 192.168.1.100
  DSMRREADER_REMOTE_DATALOGGER_NETWORK_PORT: 23
```

✅ One container
✅ No API keys
✅ Recommended for most users

---

# Server Remote Datalogger Mode

**When to use:** central server receiving data from remote forwarders.

This container:

- runs database and web UI
- does NOT read the meter
- expects incoming API data

---

## Server Configuration

```yaml
services:
  dsmr-server:
    image: xirixiz/dsmr-reader-docker:latest
    depends_on:
      - dsmrdb
    ports:
      - "80:80"
    environment:
      CONTAINER_RUN_MODE: server_remote_datalogger
      DJANGO_DATABASE_HOST: dsmrdb
      DJANGO_DATABASE_NAME: dsmrreader
      DJANGO_DATABASE_USER: dsmrreader
      DJANGO_DATABASE_PASSWORD: dsmrreader
      DJANGO_TIME_ZONE: Europe/Amsterdam
      DJANGO_SECRET_KEY: your-secret-key
      DSMRREADER_ADMIN_USER: admin
      DSMRREADER_ADMIN_PASSWORD: admin
```

---

## Setup Steps

1. Start server container
2. Access web interface
3. Navigate to Settings → API
4. Create API key
5. Copy API key for remote dataloggers

---

# Remote Datalogger Mode

**When to use:** separate device near the smart meter.

This container:

- reads the meter
- pushes telegrams to the server API
- has no database
- has no web UI

⚠️ API configuration is required.

---

## Remote Configuration (serial)

```yaml
services:
  dsmr-remote:
    image: xirixiz/dsmr-reader-docker:latest
    devices:
      - /dev/ttyUSB0:/dev/ttyUSB0
    environment:
      CONTAINER_RUN_MODE: remote_datalogger
      DSMRREADER_REMOTE_DATALOGGER_API_HOSTS: http://dsmr-server
      DSMRREADER_REMOTE_DATALOGGER_API_KEYS: your-api-key
      DSMRREADER_REMOTE_DATALOGGER_INPUT_METHOD: serial
      DSMRREADER_REMOTE_DATALOGGER_SERIAL_DEVICE: /dev/ttyUSB0
      DSMRREADER_REMOTE_DATALOGGER_SERIAL_BAUDRATE: 115200
      DSMRREADER_REMOTE_DATALOGGER_SERIAL_BYTESIZE: 8
```

---

## Remote Configuration (network meter)

```yaml
environment:
  CONTAINER_RUN_MODE: remote_datalogger
  DSMRREADER_REMOTE_DATALOGGER_API_HOSTS: http://dsmr-server
  DSMRREADER_REMOTE_DATALOGGER_API_KEYS: your-api-key
  DSMRREADER_REMOTE_DATALOGGER_INPUT_METHOD: ipv4
  DSMRREADER_REMOTE_DATALOGGER_NETWORK_HOST: 192.168.1.100
  DSMRREADER_REMOTE_DATALOGGER_NETWORK_PORT: 23
```

---

# Multi Location Setup Example

## Architecture

```
[Garage meter] → [Remote datalogger] ─┐
                                       │
                                       ├──→ [Server] → [Database] → [Web UI]
                                       │
[Vacation meter] → [Remote datalogger]─┘
```

---

# Troubleshooting

## Remote datalogger cannot connect

**Check connectivity**

```bash
curl http://dsmr-server/healthcheck
```

**Check API key**

- Verify in Settings → API
- Ensure values match

**Check logs**

```bash
docker compose logs dsmr-remote
```

---

## Server not receiving data

**Verify API enabled**

- Settings → API
- Ensure key exists

**Check firewall**

```bash
sudo ufw allow 80/tcp
```

**Check server logs**

```bash
docker compose logs dsmr-server | grep -i api
```

---

# Important Notes

- “Remote” refers to the **datalogger process location**, not the meter type
- Standalone can read meters over the network
- The remote datalogger is mainly for push based deployments
- For most home setups, **standalone + pull is simpler**
