# Running Containers as a Non-Root User

!!! warning
    Running containers as a non-root user is an advanced topic and should only be attempted after fully understanding the implications described below.

## What?

In a typical setup, the container starts with the main process running as `root`. After initialization, privileges are dropped and application services run as an unprivileged user named `app`. This design was chosen because, at the time of implementation, using a fixed non-root user at build time would have reduced flexibility and limited supported configuration options.

Although Docker now supports running containers as an arbitrary user via the `--user` flag, the images were not originally designed around this model and do not fully support it.

An alternative approach is to run Docker itself in [rootless mode](https://docs.docker.com/engine/security/rootless/). Rootless Docker uses separate user and network namespaces, meaning containers that appear to run as `root` do not have root-level access on the host. Running a container as a non-root user and running Docker rootless are distinct security models and should not be treated as equivalent.

## Why?

Some consider any container running as `root` to be an unacceptable security risk. This perspective often overestimates the practical container attack surface and misidentifies where the primary risks exist. Nevertheless, running containers as `root` can introduce risks depending on the environment.

In general, running Docker in rootless mode provides the strongest isolation model, but this is not always practical or desirable. In such cases, running individual containers as an unprivileged user can still offer meaningful security improvements.

To put the risk into perspective, consider a container exposed to the internet that allows unauthenticated access due to misconfiguration. Even if a remote code execution vulnerability is exploited and a shell is obtained inside the container, execution typically starts as an unprivileged application user. Without tools such as `sudo` and with a minimal set of installed packages, privilege escalation within the container is difficult. Even with `root` access inside the container, a separate container escape vulnerability would still be required to impact the host beyond what is already possible via mounted volumes.

Some containers require additional Linux capabilities. In such scenarios, a compromised `root` user inside the container could potentially abuse those capabilities to affect the host. For this reason, privilege and capability requirements should always be carefully evaluated.

## How?

Docker can be instructed to start a container as a specific user by providing a UID and GID:

```bash
docker run --user <uid>:<gid> …
```

Or when using Docker Compose:

```yaml
services:
  somecontainer:
    image: someimage
    user: <uid>:<gid>
```

When this approach is used, the container runs entirely as the specified user. This behavior cannot be changed without recreating the container.

Using `--user` is not a drop-in replacement for `PUID` and `PGID`.

---

## Why this is problematic for these images

These images use **s6** as a process supervisor. s6, together with the container startup logic, requires root access during initialization to perform several tasks:

- Writing runtime files to `/run`
- Creating or modifying users and groups in `/etc/passwd` and `/etc/group`
- Adjusting file ownership and permissions
- Installing packages or extracting optional components
- Allowing applications to write to their working directories

When a container is started with `--user`, none of these operations are possible because the container never runs as `root`.

---

## Limitations when using `--user`

- `PUID` and `PGID` are ignored
  The container always runs using the UID and GID specified via `--user`.

- Filesystem permissions must be managed manually
  All mounted volumes and paths must already be writable by the specified UID and GID.

- `no-new-privileges=true` cannot be enabled safely by default
  s6 requires `/run` to be writable by the user running the container. Without explicitly correcting permissions, the container will fail to start.

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

This configuration is likely to fail unless all required paths are already writable by the specified user.

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

This configuration explicitly fixes permissions for `/run` so that s6 can operate. Responsibility for all remaining permissions and compatibility concerns remains with the operator.

---

## Recommendation

For these reasons, switching existing containers to run as a non-root user is not recommended unless the implications are fully understood and the configuration has been thoroughly tested.

For most use cases, configuring `PUID` and `PGID` provides an effective balance between security, compatibility, and ease of use.

## Support Policy

Operation of this container image as a non-root user is supported on a reasonable-endeavours basis. Refer to the [Support Policy](https://xirixiz.github.io/dsmr-reader-docker-docs/misc/support-policy) for further details.
