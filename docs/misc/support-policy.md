# Support Policy

Any exceptions to this support policy are explicitly documented in the README of the relevant image.

## Formally Supported Images

Any actively maintained image from the DSMR Reader organisations is considered in scope for support. When an outdated image version is in use, upgrading to the latest available version may be required before support is provided, unless the reported issue is specific to a particular version.

Images, tags, and architectures that have been deprecated or archived are not considered in scope.

## Formally Supported Environments

DSMR Reader Docker images are built and tested using the latest stable release of Docker CE and Podman on Linux. Builds target the x86_64 (amd64) and aarch64 (arm64) architectures. Any client environment matching these characteristics is considered in scope for support. In addition, any currently supported version of Docker CE and Podman is considered supported.

Both docker compose and the Docker CLI are supported methods for managing containers, with docker compose being the preferred approach.

Running containers on Unraid is supported. Support for Unraid itself is not provided.

## Reasonable Endeavours Support

Many alternative configurations are expected to work for most images, but no guarantees are provided. A solid understanding of the underlying technologies is assumed. While these configurations are not formally supported, reasonable efforts may be made to provide assistance in the following scenarios:

- Use of DSMR Reader Docker images with alternative container runtimes
- Use of locally built DSMR Reader Docker images
- Docker Desktop on Windows, macOS, or Linux
- Docker-in-Docker setups
- Docker Swarm
- Docker installations from distribution repositories (use of the [official repositories](https://docs.docker.com/engine/install/) is recommended)
- Docker installations via Snap (use of the [official repositories](https://docs.docker.com/engine/install/) is recommended)
- End-of-life versions of Docker where upgrading is not possible
- End-of-life Linux kernel versions where upgrading is not possible
- End-of-life host Linux distributions where upgrading is not possible
- Routing container traffic through a VPN
- Use of macvlan or ipvlan networking

## Unsupported

The following configurations are entirely unsupported, and assistance is not provided even if they appear to function:

- Forks of DSMR Reader Docker maintained by third parties
- Use of third-party guides or tutorials for configuring DSMR Reader Docker
- Use of third-party container management tools such as Portainer
- Use of automated container update tools such as Watchtower
- Use of remote file storage (for example SMB or NFS) for container mounts
- Use of LXC containers
- Use of Kubernetes, including k8s or k3s

## Unsupported and Known to Be Broken

The following configurations are entirely unsupported and are known to cause severe issues or complete failure:

- Use of a custom `init` process for Docker
- Overriding container entrypoints

## Conditional Exceptions

The following configurations do not fall cleanly into a single category and are evaluated on a case-by-case basis. These configurations are not formally supported, but reasonable-endeavours assistance may be provided in some situations:

- Running containers with a root (`0`) `PUID` or `PGID`
- Running containers with a read-only container filesystem
  See: https://xirixiz.github.io/dsmr-reader-docker-docs/misc/read-only/
- Use of the `user` directive to run containers as a custom UID/GID
  See: https://xirixiz.github.io/dsmr-reader-docker-docs/misc/non-root/
