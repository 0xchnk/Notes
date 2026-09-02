# Docker Enumeration

This note provides a practical workflow for inspecting Docker containers from a host where the current user can run Docker commands. It covers container discovery, exposed ports, environment variables, networks, processes, logs, mounts, and image configuration.

> [!WARNING]
> Access to the Docker daemon often provides root-equivalent control over the host. Use these commands only in an authorized environment, and avoid modifying or restarting production containers unnecessarily.

## 1. List Running and Stopped Containers

List running containers:

```bash
docker ps
```

List every container, including stopped containers:

```bash
docker ps -a
```

Use a readable table:

```bash
docker ps -a --format 'table {{.ID}}\t{{.Names}}\t{{.Status}}\t{{.Image}}\t{{.Ports}}'
```

Important columns:

| Column | Meaning |
| --- | --- |
| `ID` | Container identifier |
| `Names` | Human-readable container name |
| `Status` | Whether the container is running, exited, restarting, or in another state |
| `Image` | Image used to create the container |
| `Ports` | Published host-to-container port mappings |

Begin with `docker ps -a`, not only `docker ps`. A stopped container can still contain valuable configuration, application data, logs, or credentials.

## 2. Identify the Container Publishing a Port

Find the running container exposing host port `8091`:

```bash
docker ps --filter publish=8091 \
  --format 'table {{.ID}}\t{{.Names}}\t{{.Status}}\t{{.Image}}\t{{.Ports}}'
```

Display all port mappings for one container:

```bash
docker port NAME
```

A published port maps a port on the Docker host to a port inside a container. Remember that a service may also listen only on a Docker network and therefore not appear as a published host port.

## 3. Inspect a Container

Display all container metadata as JSON:

```bash
docker inspect NAME
```

Display only useful state information:

```bash
docker inspect NAME --format \
  'Name={{.Name}} Status={{.State.Status}} Running={{.State.Running}} PID={{.State.Pid}} Image={{.Config.Image}}'
```

Check whether it is running:

```bash
docker inspect NAME --format '{{.State.Running}}'
```

Get the host PID of the container's main process:

```bash
docker inspect NAME --format '{{.State.Pid}}'
```

The host PID is useful when comparing what Docker reports with host-level process, namespace, or filesystem information.

## 4. Extract Environment Variables

Display every environment variable configured for a container:

```bash
docker inspect NAME --format '{{range .Config.Env}}{{println .}}{{end}}'
```

Search only for potentially sensitive variables:

```bash
docker inspect NAME --format '{{range .Config.Env}}{{println .}}{{end}}' |
grep -iE 'user|pass|secret|token|key|database|db_|url|host|port'
```

Environment variables often reveal database connection strings, service endpoints, API tokens, usernames, or passwords. They are not the only possible secret location: also check mounted configuration files, Docker secrets, application files, command arguments, and logs.

## 5. Retrieve Container IP Addresses

Simple form when the container belongs to one network:

```bash
docker inspect NAME \
  --format '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}'
```

Display every network and address:

```bash
docker inspect NAME --format \
  '{{range $name, $network := .NetworkSettings.Networks}}{{$name}} {{$network.IPAddress}}{{println}}{{end}}'
```

Inspect the default Docker bridge and all attached containers:

```bash
docker network inspect bridge
```

List Docker networks:

```bash
docker network ls
```

Containers on the same user-defined network can normally communicate using container or service names, even when their ports are not exposed on the host.

## 6. Inspect Processes and Listening Ports

List processes inside the container:

```bash
docker top NAME
```

Check listening TCP ports inside it:

```bash
docker exec NAME sh -c \
  'ss -lntp 2>/dev/null || netstat -lntp 2>/dev/null || cat /proc/net/tcp'
```

The fallback chain accounts for minimal images that do not include `ss` or `netstat`. Reading `/proc/net/tcp` is less convenient, but it can still reveal listening sockets.

## 7. Read Container Logs

Display all available logs:

```bash
docker logs NAME
```

Display only the latest 100 lines:

```bash
docker logs --tail 100 NAME
```

Follow new log entries:

```bash
docker logs -f NAME
```

Include timestamps:

```bash
docker logs --timestamps --tail 100 NAME
```

Logs may expose application errors, internal paths, usernames, tokens, database queries, or failed authentication attempts. Avoid following noisy production logs longer than necessary.

## 8. Execute Commands Inside a Container

Start a shell using `sh`:

```bash
docker exec -it NAME sh
```

Use Bash when it is installed:

```bash
docker exec -it NAME bash
```

Run one command non-interactively:

```bash
docker exec NAME id
```

Inspect the container's environment from inside:

```bash
docker exec NAME env
```

List application files:

```bash
docker exec NAME sh -c 'find /app /opt /srv -maxdepth 3 -type f 2>/dev/null'
```

The user reported by `id` is the user inside the container. It should not automatically be treated as the same user or privilege level on the Docker host.

## 9. Inspect Mounts and Volumes

Display bind mounts and named volumes:

```bash
docker inspect NAME --format \
  '{{range .Mounts}}{{println .Type .Source "->" .Destination}}{{end}}'
```

List Docker volumes:

```bash
docker volume ls
```

Inspect one volume:

```bash
docker volume inspect <VOLUME_NAME>
```

Mounted host directories may contain application source code, configuration files, databases, certificates, or credentials. A writable bind mount can also allow a container process to change files on the host, so record both the source and destination paths.

## 10. Inspect Image and Startup Configuration

Show the configured entrypoint and command:

```bash
docker inspect NAME --format \
  'Entrypoint={{json .Config.Entrypoint}} Cmd={{json .Config.Cmd}} WorkingDir={{.Config.WorkingDir}} User={{.Config.User}}'
```

Display image history:

```bash
docker history --no-trunc NAME
```

The entrypoint and command reveal what starts when the container runs. The configured user shows whether Docker attempts to run the application as a specific account; an empty value normally means the image default applies, which is commonly `root`.

Image history can reveal build commands, filenames, environment variables, or data that was deleted in a later layer. Deleting a file in a later image layer does not necessarily remove its contents from earlier layers.

## Quick Enumeration Order

For a fast first pass:

1. Run `docker ps -a` to identify all containers.
2. Inspect interesting containers with `docker inspect NAME`.
3. Review environment variables, port mappings, networks, and mounts.
4. Check processes, listening ports, application files, and logs.
5. Inspect the image's startup configuration and history.
