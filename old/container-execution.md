# Container Execution

In some situations it is necessary to inspect or interact with the internal state of a running container.

## Shell Access

Shell access is useful for debugging and inspection purposes. To open an interactive shell inside a running container, execute:

```shell
docker exec -it <container_name> /bin/bash
```

## Tailing the logs

Most images are configured to write application logs to standard output. These logs can be viewed using the docker logs command:

```shell
docker logs -f --tail=<number_of_lines_to_start_with> <container_name>
```

The --tail argument is optional. When omitted, the command outputs all available logs, which may be excessive for long-running containers.

For convenience, the following Bash alias can be used to simplify log access:

```shell
# ~/.bash_aliases
alias dtail='docker logs -tf --tail="50" "$@"'
```

Logs can then be followed using:

```shell
dtail <container_name>
```

## Checking the build version

When investigating issues, identifying the exact image version in use is important. Problems may already be resolved in newer releases, or the observed behavior may indicate a newly introduced issue.

To display version and build information for a running container (for example, dsmr):

```shell
docker inspect -f '{{ with .Config.Labels -}}
DSMR upstream version: {{ index . "io.github.dsmrreader.upstream.version" }}
DSMR Docker release:   {{ index . "io.github.dsmrreader.docker.release" }}
OCI version:           {{ index . "org.opencontainers.image.version" }}
Build date:            {{ index . "org.opencontainers.image.build_date" }}
Revision:              {{ index . "org.opencontainers.image.revision" }}
{{- end }}' <container_name>
```

An alternative method to retrieve version information from the running container:

```shell
docker exec -ti <container_name> bash -c 'cat /build_version'
```

To inspect version information directly from an image (for example, ghcr.io/xirixiz/dsmr-reader-docker):

```shell
docker inspect -f '{{ with .Config.Labels -}}
DSMR upstream version: {{ index . "io.github.dsmrreader.upstream.version" }}
DSMR Docker release:   {{ index . "io.github.dsmrreader.docker.release" }}
OCI version:           {{ index . "org.opencontainers.image.version" }}
Build date:            {{ index . "org.opencontainers.image.build_date" }}
Revision:              {{ index . "org.opencontainers.image.revision" }}
{{- end }}' <image_name>
```
