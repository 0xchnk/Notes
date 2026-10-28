# Docker Enumeration

## 1. List Running and Stopped Containers

**Run on the Docker Host.**

List running containers:

```bash
docker ps
```

List every container, including stopped containers:

```bash
docker ps -a
```

Display the results in a readable table:

```bash
docker ps -a --format 'table {{.ID}}\t{{.Names}}\t{{.Status}}\t{{.Image}}\t{{.Ports}}'
```

Important columns:

|Column|Meaning|
|---|---|
|`ID`|Container identifier|
|`Names`|Human-readable container name|
|`Status`|Running, exited, restarting, and so on|
|`Image`|Image used to create the container|
|`Ports`|Published host-to-container port mappings|

Start by identifying running and stopped containers, their images, and any exposed ports. Container names and images can provide useful clues about the applications and services running on the host.

## 2. Identify the Container Publishing a Port

**Run on the Docker Host.**

Find running containers publishing host port `8091`:

```bash
docker ps --filter publish=8091 \
  --format 'table {{.ID}}\t{{.Names}}\t{{.Status}}\t{{.Image}}\t{{.Ports}}'
```

Display all port mappings for a specific container:

```bash
docker port NAME
```

Replace `NAME` with the container name or ID.

Port mappings help identify which container serves a particular host port and which internal container port receives the traffic.

## 3. Inspect a Container

**Run on the Docker Host.**

Display all container metadata as JSON:

```bash
docker inspect NAME
```

Display the most useful state information:

```bash
docker inspect NAME --format \
  'Name={{.Name}} Status={{.State.Status}} Running={{.State.Running}} PID={{.State.Pid}} Image={{.Config.Image}}'
```

Check whether the container is running:

```bash
docker inspect NAME --format '{{.State.Running}}'
```

Get the host PID of the container's main process:

```bash
docker inspect NAME --format '{{.State.Pid}}'
```

The host PID can help correlate a container with processes visible from the Docker host. Remember that container process IDs and host process IDs can differ because of PID namespaces.

## 4. Extract Environment Variables

**Run on the Docker Host.**

Display every environment variable configured for a container:

```bash
docker inspect NAME --format '{{range .Config.Env}}{{println .}}{{end}}'
```

Search for potentially sensitive variables:

```bash
docker inspect NAME --format '{{range .Config.Env}}{{println .}}{{end}}' |
grep -iE 'user|pass|secret|token|key|database|db_|url|host|port'
```

Environment variables may reveal database credentials, API tokens, service endpoints, usernames, or other configuration details.

They are not the only possible source of sensitive information. Also inspect mounted configuration files, Docker secrets, application files, command arguments, and logs where relevant.

## 5. Retrieve Container IP Addresses

**Run on the Docker Host.**

Display the container IP addresses:

```bash
docker inspect NAME \
  --format '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}'
```

This compact form works for quick checks, but it does not identify which network each address belongs to.

Display every attached network and its IP address:

```bash
docker inspect NAME --format \
  '{{range $name, $network := .NetworkSettings.Networks}}{{$name}} {{$network.IPAddress}}{{println}}{{end}}'
```

List Docker networks:

```bash
docker network ls
```

Inspect the default Docker bridge and its attached containers:

```bash
docker network inspect bridge
```

Container IP addresses are useful when mapping communication between containers or identifying services that may be reachable from another container or the host.

## 6. Inspect Processes and Listening Ports

**Run on the Docker Host.**

List processes running inside a container:

```bash
docker top NAME
```

Check listening TCP ports from inside the container:

```bash
docker exec NAME sh -c \
  'ss -lntp 2>/dev/null || netstat -lntp 2>/dev/null || cat /proc/net/tcp'
```

The first command lists processes. The second attempts to identify listening TCP sockets using `ss`, falls back to `netstat`, and finally reads `/proc/net/tcp` if neither utility is available.

The `/proc/net/tcp` fallback is less readable and does not provide the same process information as `ss -lntp`.

## 7. Read Container Logs

**Run on the Docker Host.**

Display all available logs:

```bash
docker logs NAME
```

Display only the latest 100 lines:

```bash
docker logs --tail 100 NAME
```

Follow new log entries in real time:

```bash
docker logs -f NAME
```

Include timestamps:

```bash
docker logs --timestamps --tail 100 NAME
```

Logs can reveal application errors, service startup details, internal endpoints, authentication failures, and occasionally sensitive information.

## 8. Execute Commands Inside a Container

**Run these commands on the Docker Host.** The commands themselves execute inside the specified container.

Start an interactive shell using `sh`:

```bash
docker exec -it NAME sh
```

Use Bash if it is installed:

```bash
docker exec -it NAME bash
```

Run a single command without opening an interactive shell:

```bash
docker exec NAME id
```

Inspect the environment from inside the container:

```bash
docker exec NAME env
```

List application files in common directories:

```bash
docker exec NAME sh -c 'find /app /opt /srv -maxdepth 3 -type f 2>/dev/null'
```

The `id` command reveals the identity under which the command executes inside the container. Check the user's privileges and accessible files when assessing the container's security boundaries.

## 9. Inspect Mounts and Volumes

**Run on the Docker Host.**

Display bind mounts and named volumes attached to a container:

```bash
docker inspect NAME --format \
  '{{range .Mounts}}{{println .Type .Source "->" .Destination}}{{end}}'
```

List Docker volumes:

```bash
docker volume ls
```

Inspect a specific volume:

```bash
docker volume inspect <VOLUME_NAME>
```

Mount information shows which host paths or Docker-managed volumes are accessible inside the container. These may contain application source code, configuration files, databases, certificates, or credentials.

Pay particular attention to sensitive host directories mounted into a container and whether the mount is read-write or read-only.

## 10. Inspect Image and Startup Configuration

**Run on the Docker Host.**

Display the configured entrypoint, command, working directory, and user:

```bash
docker inspect NAME --format \
  'Entrypoint={{json .Config.Entrypoint}} Cmd={{json .Config.Cmd}} WorkingDir={{.Config.WorkingDir}} User={{.Config.User}}'
```

Display the image's build history:

```
docker history --no-trunc NAME
```

The startup configuration shows which command runs when the container starts and which user is configured to run it.

Image history may reveal build commands, filenames, environment variables, or other configuration details. Files deleted in later image layers may still be recoverable from earlier layers, depending on how the image was built.

**— 0xchnk**
