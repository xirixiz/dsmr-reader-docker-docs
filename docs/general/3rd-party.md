# Platform Specific Setup

## Raspberry Pi

### Prerequisites

```bash
# Update system
sudo apt-get update
sudo apt-get upgrade

# Install dependencies
sudo apt-get install docker.io docker-compose-plugin git
```

💡 Alternatively, install Docker via the official Docker repository for newer versions.

---

### Recommended settings

```yaml
services:
  dsmr:
    # Use specific version for stability
    image: xirixiz/dsmr-reader-docker:6.2.0

    # Optional resource limits for older Pi models
    mem_limit: 512m
    cpus: 1.0
```

**Storage consideration**

- Prefer an external USB drive for the database volume
- SD cards wear out quickly with database workloads

---

## Synology NAS

### Install Container Manager

1. Open **Package Center**
2. Install **Container Manager** (or Docker on older DSM)
3. Deploy your compose stack

---

### USB Serial driver

Install **SynoKernel USB Serial Drivers** from SynoCommunity.

⚠️ Compatibility depends on DSM version and NAS model.
⚠️ On some DSM 7 systems USB serial support may be limited.

---

### Serial device path

```bash
# Find device
ls -l /dev/ttyUSB*

# Temporary permission fix
sudo chmod 666 /dev/ttyUSB0
```

---

## Windows (Docker Desktop)

### Prerequisites

- Windows 10/11 Pro, Enterprise, or Education
- WSL2 enabled
- Docker Desktop installed

---

### Notes on USB serial

Direct USB passthrough is **not supported** in Docker Desktop.

Options:

1. Use network smart meter (recommended)
2. Use USB/IP via `usbipd`
3. Run DSMR Reader on native Linux

---

## macOS

### Install Docker Desktop

Download from:

https://www.docker.com/products/docker-desktop

Or via Homebrew:

```bash
brew install --cask docker
```

---

### Serial device

macOS devices typically appear as:

```
/dev/cu.usbserial-*
/dev/tty.usbserial-*
```

Prefer the `cu.*` devices.

Example:

```yaml
devices:
  - /dev/cu.usbserial-AB0IXYZ:/dev/ttyUSB0
```

⚠️ USB passthrough depends on Docker Desktop access to the device and may not work on all systems.

---

## HomeWizard P1 Meter (Advanced / Experimental)

⚠️ This is an advanced workaround and not the recommended primary setup.
Whenever possible, prefer:

- direct USB
- ser2net
- P1 TCP gateway

---

### Enable HomeWizard Local API

Enable the **Local API** in the HomeWizard app.

---

### Example plugin

Create `plugins/homewizard_p1.py`:

```python
import logging
import requests
from django.dispatch import receiver
from dsmr_backend.signals import backend_called
from dsmr_datalogger.services.datalogger import telegram_to_reading

HOMEWIZARD_ENDPOINT = 'http://1.2.3.4:80/api/v1/telegram'  # adjust IP
HOMEWIZARD_TIMEOUT = 5

logger = logging.getLogger(__name__)

@receiver(backend_called)
def handle_backend_called(**kwargs):
    try:
        response = requests.get(HOMEWIZARD_ENDPOINT, timeout=HOMEWIZARD_TIMEOUT)
        response.raise_for_status()
    except requests.exceptions.RequestException as e:
        logger.error(f'HomeWizard plugin: failed to retrieve telegram: {e}')
        return

    try:
        telegram_to_reading(data=response.text)
    except Exception as e:
        logger.exception(f'HomeWizard plugin: failed to process telegram: {e}')
```

---

### Compose configuration

```yaml
services:
  dsmr:
    image: xirixiz/dsmr-reader-docker:latest
    volumes:
      - ./plugins/homewizard_p1.py:/app/dsmr_plugins/modules/homewizard_p1.py:ro
    environment:
      CONTAINER_RUN_MODE: standalone
      DSMRREADER_PLUGINS: dsmr_plugins.modules.homewizard_p1
```

---

### Verification

```bash
docker compose logs dsmr | grep -i homewizard
```

---

### References

- Original discussion: https://github.com/xirixiz/dsmr-reader-docker/issues/301
- HomeWizard API: https://api-documentation.homewizard.com/docs/v1/telegram/
