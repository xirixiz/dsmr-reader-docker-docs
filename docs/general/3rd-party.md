# Platform Specific Setup

## Raspberry Pi

**Prerequisites:**
```bash
# Update system
sudo apt-get update
sudo apt-get upgrade

# Install dependencies
sudo apt-get install docker.io docker compose git
```

**Recommended settings:**
```yaml
services:
  dsmr:
    # Use specific version for stability
    image: xirixiz/dsmr-reader-docker:6.2.0

    # Limit resources on Raspberry Pi 3 or older
    mem_limit: 512m
    cpus: 1.0
```

**Storage consideration:**
- Use external USB drive for database volume
- SD cards wear out with database writes

## Synology NAS

**Install Docker package:**
1. Open Package Center
2. Install "Docker" package
3. Install "Container Manager"

**USB Serial driver:**
Install `synokernel-usbserial` from Community Package Center

**Create stack via Container Manager:**
- Copy docker-compose.yaml content
- Adjust paths for Synology structure

**Serial device path:**
```bash
# Find device
ls -l /dev/ttyUSB*

# Set permissions
sudo chmod 666 /dev/ttyUSB0
```

## Windows (Docker Desktop)

**Prerequisites:**
- Windows 10/11 Pro, Enterprise, or Education
- WSL2 enabled
- Docker Desktop for Windows

**Installation:**
1. Install Docker Desktop
2. Enable WSL2 backend
3. Create project in WSL2 Linux distribution

**Serial device access:**
- Direct USB passthrough not supported in WSL2
- Options:
  1. Use network smart meter (HomeWizard)
  2. Use USB/IP forwarding
  3. Run on native Linux instead

## macOS

**Prerequisites:**
```bash
# Install Docker Desktop for Mac
# Download from: https://www.docker.com/products/docker-desktop

# Or via Homebrew
brew install --cask docker
```

**Serial device:**
```bash
# macOS devices appear as:
/dev/cu.usbserial-*
/dev/tty.usbserial-*

# Use cu.* devices
ls -l /dev/cu.*
```

**docker-compose.yaml:**
```yaml
devices:
  - /dev/cu.usbserial-AB0IXYZ:/dev/ttyUSB0
```

## HomeWizard P1 Meter Integration

1. **Enable HomeWizard Local API** in the HomeWizard app

2. **Create plugin file** `plugins/homewizard_p1.py`:

```python
import logging
import requests
from django.dispatch import receiver
from dsmr_backend.signals import backend_called
from dsmr_datalogger.services.datalogger import telegram_to_reading

HOMEWIZARD_ENDPOINT = 'http://1.2.3.4:80/api/v1/telegram'  # Replace with your IP
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

3. **Update docker-compose.yaml**:

```yaml
services:
  dsmr:
    image: xirixiz/dsmr-reader-docker:latest
    volumes:
      - ./plugins/homewizard_p1.py:/app/dsmr_plugins/modules/homewizard_p1.py:ro
    environment:
      CONTAINER_RUN_MODE: server_remote_datalogger
      DSMRREADER_PLUGINS: dsmr_plugins.modules.homewizard_p1
      # ... other environment variables ...
```

4. **Restart containers**:

```bash
docker compose down && docker compose up -d
```

### Verification

Check plugin is loading:
```bash
docker compose logs dsmr | grep -i homewizard
```

### References

- [Original GitHub Discussion](https://github.com/xirixiz/dsmr-reader-docker/issues/301)
- [Home Assistant Alternative](https://community.home-assistant.io/t/dsmr-reader-docker-and-homewizard-p1-meter-integration/747265)
