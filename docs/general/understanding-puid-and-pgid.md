# Understanding PUID and PGID

!!! info
  At this time, DSMR Reader Docker does not fully support running with Docker’s --user flag.

  While recent Docker versions allow containers to run under an arbitrary UID and GID using `--user`, DSMR Reader relies on internal user and group handling, filesystem permissions, and startup logic that assume the container manages its own user context. Using `--user` can therefore lead to permission errors, failed startups, or subtle runtime issues.

  For this reason, we recommend continuing to use `PUID` and `PGID` to control file ownership and access:
  - They integrate cleanly with the image’s internal permission model
  - They ensure correct access to mounted volumes
  - They avoid unpredictable behaviour caused by overriding the container user

  Support for --user may be added in the future, but until then it is not considered a supported configuration.

## Why use these?

By default, Docker starts containers with processes running as the `root` user. This is necessary for certain low level operations such as network setup, process management, and interacting with the filesystem. As a result, applications inside the container also run with elevated privileges.

Running applications as `root` is not ideal for everyday use. It increases the impact of potential misconfigurations or vulnerabilities and can grant applications more access than strictly necessary. While careful volume and port mappings can reduce risk, they do not eliminate it entirely.

Another common issue is file ownership on mounted volumes. When a container runs as `root`, any files or directories it creates on mapped host volumes will also be owned by `root`, which can make them difficult or impossible for the host user to manage without additional permission changes.

To address this, our images support the use of `PUID` and `PGID`. These environment variables map the container’s internal application user to a specific user and group on the host system. This ensures that files created within mounted volumes are owned by the correct host user and that the container runs with the minimum privileges required.

The container is designed to use this model and should be configured with `PUID` and `PGID` accordingly.

## Using the variables

When creating a container from the container image, ensure you use the `PUID` and `PGID` options:


For use with `docker-compose`, add them to the `environment:` section:

```yaml
environment:
  - PUID=1000
  - PGID=1000
```

In most cases, you will want to use your own user ID. You can find this by running the command below. The relevant values are the `uid` and `gid`.

```shell
id $user
```
