# Running Containers as a Non-Root User

!!! warning
    Running containers as a non-root user is an advanced topic and should only be attempted if you fully understand the implications described below.

## What?

In a typical setup, our images start with the container running as `root`. After initialization, privileges are dropped and the application services run as an unprivileged user named `app`. This approach was chosen because, at the time, using a fixed non-root user at build time would have limited the flexibility and configuration options we wanted to support.

While Docker now allows containers to run as an arbitrary user via the `--user` flag, our images were not originally designed with this model in mind and do not fully support it yet.

An alternative approach is to run [Docker itself in rootless mode](https://docs.docker.com/engine/security/rootless/). Rootless Docker uses separate user and network namespaces, meaning that even containers that appear to run as `root` do not have root-level access on the host. Running a container as a non-root user and running Docker rootless are different security models and are often incorrectly treated as equivalent.

## Why?

Some people argue that a container running as `root` under any circumstances is an unacceptable security risk. This view often overstates the real container attack surface and misunderstands where the actual risks lie. That said, running containers as `root` can introduce risks depending on the environment.

In general, the strongest approach is to run Docker itself in rootless mode, but this is not always practical or desirable. In those cases, running individual containers as an unprivileged user can still provide meaningful security benefits.

To put the risk into perspective, consider a container that is exposed to the internet and, due to misconfiguration, allows unauthenticated access. Even if an attacker discovers a remote code execution vulnerability and gains a shell inside the container, they typically start as an unprivileged application user. Without tools like sudo and with a minimal set of installed packages, escalating privileges inside the container is difficult. Even with `root` access inside the container, an attacker would still need a separate container escape vulnerability to impact the host beyond what is already possible through mounted volumes.

It is worth noting that some containers require additional Linux capabilities. In such cases, a compromised `root` user inside the container could potentially abuse those capabilities to affect the host, which is why privilege and capability requirements should always be carefully considered.

## How?
You can force Docker to start a container as a specific user by supplying a `UID` and `GID`:

```bash
docker run --user <uid>:<gid> …
```

or in Docker Compose:

```yaml
services:
  somecontainer:
    image: someimage
    user: <uid>:<gid>
```

When you do this, **the container runs entirely as that user**. This cannot be changed without recreating the container.

However, using `--user` is not a drop-in replacement for `PUID` and `PGID`.

---

## Why this is problematic for our images

Our images use **s6** as a process supervisor. s6, along with the startup logic of the image, expects to perform several operations that require root access during container initialization:

- Writing runtime files to `/run`
- Creating or modifying users and groups in `/etc/passwd` and `/etc/group`
- Adjusting ownership and permissions
- Installing packages or extracting optional components
- Allowing applications to write to their working directories

When the container is started with `--user`, none of this can happen because the container never runs as root.

---

## Limitations when using `--user`

- `PUID` and `PGID` are ignored
  The container will always run using the UID and GID specified via `--user`.

- You must manage all filesystem permissions manually
  Any mounted volumes or paths must already be writable by the specified UID and GID.

- `no-new-privileges=true` cannot be used safely by default
  s6 requires `/run` to be writable by the user running the container. Without explicitly fixing permissions, the container will fail to start.

---

## Example configurations

### Running with `--user` only (**not recommended**)

```yaml
services:
  dsmr:
    image: ghcr.io/xirixiz/dsmr-reader-docker
    container_name: dsmr
    user: 1000:1000
    ports:
      - "7777:80"
      - "7779:443"
    restart: unless-stopped
```

This setup will likely fail unless all required paths are already writable.

---

### Running with `--user` and required workarounds

```yaml
services:
  dsmr:
    image: ghcr.io/xirixiz/dsmr-reader-docker
    container_name: dsmr
    user: 1000:1000
    ports:
      - "7777:80"
      - "7779:443"
    restart: unless-stopped
    tmpfs:
      - /run:uid=1000,gid=1000,exec
    security_opt:
      - no-new-privileges=true
```

This explicitly fixes `/run` permissions so s6 can function, but **you are still responsible for all other permissions and compatibility issues**.

---

## Recommendation

For these reasons, we **do not recommend switching existing containers to run as a non-root user** unless you fully understand the implications and have tested the setup thoroughly.

For most users, using `PUID` and `PGID` provides the best balance of security, compatibility, and ease of use.


## Support Policy

Operation of our container image with a non-root user is supported on a Reasonable Endeavours basis. Please see our [Support Policy](https://misc/supportpolicy) for more details.

## Change History


