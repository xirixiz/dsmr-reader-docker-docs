---
hide:
  - navigation
  - toc
---
# Frequently Asked Questions

??? faq "My host filesystem is incompatible with my docker storage driver"

    ##### My host filesystem is incompatible with my docker storage driver { #storage }

    === "Description"

        Some host filesystem types are not compatible with Docker’s default storage driver (`overlay2`).

    === "Symptoms"

        Errors similar to the following may appear in container logs:

        ```text
        ERROR Found no accessible config files
        ```

        or

        ```text
        Directory not empty. This directory contains an empty ignorecommands sub-directory
        ```

    === "Resolution"

        Refer to the supported backing filesystems in the Docker documentation:
        https://docs.docker.com/storage/storagedriver/select-storage-driver/#supported-backing-filesystems

        For a host filesystem of ZFS, the Docker storage driver must be set to `zfs`. For a host filesystem of Btrfs, the Docker storage driver must be set to `btrfs`.

        Correcting the storage driver/filesystem mismatch resolves the issue. Changes inside a container do not resolve this type of problem.

??? faq "I want to reverse proxy an application which defaults to https with a self-signed certificate"

    ##### Reverse proxy to an HTTPS backend using a self-signed certificate { #strict-proxy }

    === "Traefik"

        Configure a `serversTransport` rule that disables certificate verification, then apply it to the service. Also set the backend scheme to `https`.

        Create a [ServerTransport](https://doc.traefik.io/traefik/routing/services/#serverstransport_1) in the dynamic Traefik configuration (example name: `ignorecert`):

        ```yml
        http:
          serversTransports:
            ignorecert:
              insecureSkipVerify: true
        ```

        Apply the rule to the `foo` service and specify HTTPS on the backend:

        ```yml
        - traefik.http.services.foo.loadbalancer.serverstransport=ignorecert@file
        - traefik.http.services.foo.loadbalancer.server.scheme=https
        ```

    === "Caddy"

        When reverse proxying an HTTPS backend that uses a self-signed certificate, Caddy rejects it by default because the certificate authority cannot be verified.

        Disable verification in the site entry of the [Caddyfile](https://caddyserver.com/docs/quick-starts/caddyfile) as shown below.

        !!! note
            Replace `dsmrreader.xxx.com` with the intended domain and replace `172.xxx.xxx.xxx:7779` with the backend service IP and port.

        ```caddyfile
        dsmrreader.xxx.com {
            reverse_proxy https://172.xxx.xxx.xxx:7779 {
                transport http {
                    tls
                    tls_insecure_skip_verify
                }
            }
        }
        ```

        ???+ tip "Bonus Tip 1: Caddy Snippets"
            If multiple services require the same configuration, define a reusable [Caddy snippet](https://caddyserver.com/docs/caddyfile/concepts#snippets):

            ```caddyfile
            (allow_insecure_ssl) {
                transport http {
                    tls
                    tls_insecure_skip_verify
                }
            }
            dsmrreader.xxx.com {
                reverse_proxy https://172.xxx.xxx.xxx:7779 {
                    import allow_insecure_ssl
                }
            }
            ```

        ???+ tip "Bonus Tip 2: caddy-docker-proxy"
            When using [caddy-docker-proxy](https://github.com/lucaslorentz/caddy-docker-proxy), apply the following labels in `docker-compose.yml`:

            ```yaml
            labels:
              caddy: dsmrreader.xxx.com
              caddy.reverse_proxy: "{{upstreams https 7779}}"
              caddy.reverse_proxy.transport: http
              caddy.reverse_proxy.transport.tls:
              caddy.reverse_proxy.transport.tls_insecure_skip_verify:
            ```

??? faq "Why is docker-compose recommended over Portainer?"

    ##### Why is docker-compose recommended over Portainer? { #portainer }

    Portainer introduces several issues that complicate troubleshooting and support, including:

    - Advanced settings can be hidden or unavailable
    - Mount source/target order can be applied incorrectly
    - Inconsistent case-sensitivity behavior
    - Custom networks for inter-container communication may not be created automatically
    - Compose implementations can differ across architectures
    - Environment variables can be applied incorrectly during container upgrades

??? faq "Inexplicable issues when running ubuntu"

    ##### Inexplicable issues when running Ubuntu { #snap }

    === "Description"

        Some setups encounter issues with containers despite correct configuration and unremarkable logs. A frequent root cause is installation of Docker via Snap.

    === "Symptoms"

        Symptoms can be difficult to identify. On Ubuntu systems where everything appears correct, verify whether Docker is installed via Snap.

    === "Resolution"

        Confirm the Ubuntu version:

        ```bash
        lsb_release -a
        ```

        Example output:

        ```text
        No LSB modules are available.
        Distributor ID: Ubuntu
        Description:    Ubuntu 22.04.3 LTS
        Release:        22.04
        Codename:       jammy
        ```

        Check whether Docker is installed via Snap:

        ```bash
        snap list | grep docker
        ```

        Example output:

        ```text
        docker  20.10.24       2904   latest/stable  canonical**  -
        ```

        If Snap Docker is installed, it can coexist with a repository-based installation and may take precedence. Remove the Snap installation:

        ```bash
        sudo snap remove docker
        ```

        !!! info
            Unless automatic snapshots are disabled, a snapshot of Snap data is saved upon removal and can be restored with `snap restore`. The `--purge` option disables automatic snapshots.

        Confirm remaining Docker paths:

        ```bash
        sudo whereis docker
        ```

        Expected output after Snap removal typically resembles:

        ```text
        docker: /usr/libexec/docker
        ```

        If both repository-based and Snap installations coexist, output may include Snap paths such as:

        ```text
        docker: /usr/bin/docker /etc/docker /usr/libexec/docker /snap/bin/docker.machine /snap/bin/docker.help /snap/bin/docker.compose /snap/bin/docker /usr/share/man/man1/docker.1.gz
        ```

        After Snap removal, follow the official installation instructions if Docker is not present:
        https://docs.docker.com/engine/install/ubuntu/

        If commands fail with `-bash: /snap/bin/docker: No such file or directory`, start a new shell or clear the shell hash table:

        ```bash
        hash -r
        ```

        Version information may be verified with:

        ```bash
        docker --version && docker compose version
        ```
