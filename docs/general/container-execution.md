# Container Execution

You may find at some point you need to view the internal data of a container.

## Shell Access

Particularly useful when debugging the application - to shell in to one of our containers, run the following:

```shell
docker exec -it <container_name> /bin/bash
```

## Tailing the logs

The vast majority of our images are configured to output the application logs to the console, which in Docker's terms means you can access them using the `docker logs` command:

```shell
docker logs -f --tail=<number_of_lines_to_start_with> <container_name>
```

The `--tail` argument is optional, but useful if the application has been running for a long time - the `logs` command by default will output _all_ logs.

To make life simpler for yourself here's a handy bash alias to do some of the leg work for you:

```shell
# ~/.bash_aliases
alias dtail='docker logs -tf --tail="50" "$@"'
```

Execute it with `dtail <container_name>`.

## Checking the build version

If you are experiencing issues, knowing exactly which image version your container is running helps us investigate more effectively. You may be reporting a problem that has already been identified and fixed in a newer release. If you are already running the latest version, it may indicate a newly discovered issue, which we would definitely like to investigate further.

To obtain the build version for the container (f.e. 'dsmr'):

```shell
docker inspect -f '{{ with .Config.Labels -}}
DSMR upstream:  {{ index . "io.github.dsmrreader.upstream.version" }}
Docker release: {{ index . "io.github.dsmrreader.docker.release" }}
OCI version:    {{ index . "org.opencontainers.image.version" }}
Build date:     {{ index . "org.opencontainers.image.build_date" }}
Revision:       {{ index . "org.opencontainers.image.revision" }}
{{- end }}' <container_name>

```

Or the image (f.e 'ghcr.io/xirixiz/dsmr-reader-docker'):

```shell
docker inspect -f '{{ with .Config.Labels -}}
DSMR upstream:  {{ index . "io.github.dsmrreader.upstream.version" }}
Docker release: {{ index . "io.github.dsmrreader.docker.release" }}
OCI version:    {{ index . "org.opencontainers.image.version" }}
Build date:     {{ index . "org.opencontainers.image.build_date" }}
Revision:       {{ index . "org.opencontainers.image.revision" }}
{{- end }}' <image_name>
```
