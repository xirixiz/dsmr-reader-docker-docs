---
hide:
  - navigation
---
# Frequently Asked Questions

??? faq "My host filesystem is incompatible with my docker storage driver"

    ##### My host filesystem is incompatible with my docker storage driver { #storage }

    === "Description"

        Some host file systems types are not compatible with the default storage driver of docker (overlay2)

    === "Symptoms"

        If your host is affected you may see errors in your containers such as:

        ```text
        ERROR Found no accessible config files
        ```

        or

        ```text
        Directory not empty. This directory contains an empty ignorecommands sub-directory
        ```

    === "Resolution"

        As shown in [Docker docs](https://docs.docker.com/storage/storagedriver/select-storage-driver/#supported-backing-filesystems)

        A host filesystem of zfs requires a docker storage driver of zfs and a host file system of btrfs requires a docker storage driver of btrfs.
        Correcting this oversight will resolve the issue. This is not something that a container change will resolve.

??? faq "I want to reverse proxy which defaults to https with a self-signed certificate"

    ##### I want to reverse proxy an application which defaults to https with a self-signed certificate { #strict-proxy }

    === "Traefik"

        In this example, we will configure a serverTransport rule we can apply to a service, as well as telling Traefik to use https on the backend for the service.

        Create a [ServerTransport](https://doc.traefik.io/traefik/routing/services/#serverstransport_1) in your dynamic Traefik configuration; we are calling ours `ignorecert`.

        ```yml
            http:
                serversTransports:
                    ignorecert:
                    insecureSkipVerify: true
        ```

        Then on our `foo` service we tell it to use this rule, as well as telling Traefik the backend is running on https.

        ```yml
            - traefik.http.services.foo.loadbalancer.serverstransport=ignorecert@file
            - traefik.http.services.foo.loadbalancer.server.scheme=https
        ```

    === "Caddy"

        When reverse proxying an HTTPS backend that uses a self-signed certificate, Caddy will normally reject it because it cannot verify the certificate authority.

        To skip this verification we can modify site entry of the [caddyfile](https://caddyserver.com/docs/quick-starts/caddyfile) as shown below:

    !!! note
        Replace `dsmrreader.xxx.com` with your domain and `172.xxx.xxx.xxx:7779` with your backend service IP and port.


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
        If you find yourself needing to do this for multiple services, you can also define a [caddy snippet](https://caddyserver.com/docs/caddyfile/concepts#snippets) and reuse it in your caddyfile like so:

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
        If you use [caddy-docker-proxy](https://github.com/lucaslorentz/caddy-docker-proxy), you can simply apply the following labels to your docker-compose yaml file:

        ```yaml
        labels:
            caddy: dsmrreader.xxx.com
            caddy.reverse_proxy: "{{upstreams https 7779}}"
            caddy.reverse_proxy.transport: http
            caddy.reverse_proxy.transport.tls:
            caddy.reverse_proxy.transport.tls_insecure_skip_verify:
        ```

??? faq "Why do we recommend to use docker-compose over Portainer?"

    ##### Why do we recommend to use docker-compose over Portainer? { #portainer }

    Portainer has many issues which make it hard for us to support, such as:

    - Advanced settings are hidden and some aren't available at all
    - Incorrect order of source and target of mounts
    - Inconsistent case-sensitivity
    - No automatically created custom networks for inter-container communication
    - Inconsistent compose implementations on different architectures
    - Incorrectly applying environment variables on container upgrades

??? faq "Inexplicable issues when running ubuntu"

    ##### Inexplicable issues when running ubuntu { #snap }

    === "Description"

        Many users have been facing issues that are simply inexplicable. The logs show no problems, the compose is fine, eventually it turns out they've installed the SNAP version of docker which is the source of the issues.

    === "Symptoms"

        It's difficult to identify the symptoms, but if you are running ubuntu and believe you have done everything correctly, check for SNAP docker.

    === "Resolution"

        First the user must be on an appropriate version of ubuntu to face this issue (as far as I am aware)

        `lsb_release -a` would result in something similar to the below output
        ```bash
        No LSB modules are available.
        Distributor ID: Ubuntu
        Description:    Ubuntu 22.04.3 LTS
        Release:        22.04
        Codename:       jammy
        ```

        `snap list | grep docker` would result in something similar to the below output
        ```bash
        docker  20.10.24       2904   latest/stable  canonical**  -
        ```

        This means the snap version of docker is installed. Unfortunately, even if the user installed docker from the proper repo, this snap version will coexist AND be preferred. They will need to remove it, as shown below.

        ```bash
        xirixiz@home-server:~/$ sudo snap remove docker
        [sudo] password for xirixiz:
        2026-01-11T01:06:26Z INFO Waiting for "snap.docker.dockerd.service" to stop.
        docker removed
        xirixiz@home-server:~/$
        ```

        !!! info
            Unless automatic snapshots are disabled, a snapshot of all data for the snap is saved upon removal, which is then available for future restoration with snap restore. The --purge option disables automatically creating snapshots.

        Following this, confirm nothing related to snap still shows.
        ```bash
        ~$ sudo whereis docker
        docker: /usr/libexec/docker
        ```
        above is what we might want to see, below is how it would look if both official AND snap are installed. Seeing the snap stuff removed but the official there is OK.
        ```bash
        ~$ sudo whereis docker

        docker: /usr/bin/docker /etc/docker /usr/libexec/docker /snap/bin/docker.machine /snap/bin/docker.help /snap/bin/docker.compose /snap/bin/docker /usr/share/man/man1/docker.1.gz
        ```
        As you can see in the second one, multiple versions can coexist which is a big tshoot problem.

        Once this is complete, if the expected version isn't present, simply follow [docker install on ubuntu](https://docs.docker.com/engine/install/ubuntu/)

        When they finish, running `docker` commands may result in `-bash: /snap/bin/docker: No such file or directory` if this is the case, this is simply a shell patch issue, they can launch a new shell or simply input `hash -r` which should resolve the problem. Version info at the time of this writing should be
        ```bash
        ~ # docker --version && docker compose version
        Docker version 29.1.1, build 0aedba5
        Docker Compose version v2.40.3
        ```
