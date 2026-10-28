# SSH Access, Tunneling, and File Transfers

SSH provides encrypted remote shell access, but it can also forward TCP connections and transfer files. This note covers normal SSH access, public-key authentication, local and remote port forwarding, combined tunnels, and SCP.

## 1. Normal SSH Login

**Run on the Attacker machine.**

Connect to the target using a username and password:

```bash
ssh <USER>@TARGET_IP
```

Connect to a non-default SSH port:

```bash
ssh -p <SSH_PORT> <USER>@TARGET_IP
```

Useful options for unstable connections:

```bash
ssh -o ServerAliveInterval=30 -o ServerAliveCountMax=3 <USER>@TARGET_IP
```

- `ServerAliveInterval=30` sends a keepalive message every 30 seconds while no data is received.
- `ServerAliveCountMax=3` closes the connection after three unanswered keepalive messages.

These options help detect dead connections, especially when working through VPNs, pivots, or unstable lab networks.

## 2. SSH Public-Key Authentication

### Generate a Key Pair

**Run on the Attacker machine.**

Generate an Ed25519 key pair:

```bash
ssh-keygen -t ed25519 -a 100 -f ~/.ssh/lab_ed25519 -C "lab access"
```

This creates two files:

```bash
~/.ssh/lab_ed25519      # Private key — keep it on the Attacker
~/.ssh/lab_ed25519.pub  # Public key — install it on the Target
```

Protect the private key:

```bash
chmod 600 ~/.ssh/lab_ed25519
```

The private key stays on the Attacker machine. The public key is added to the intended user's `~/.ssh/authorized_keys` file on the Target.

### Method A: Install the Key with `ssh-copy-id`

**Run on the Attacker machine**, assuming password-based SSH access already works.

```bash
ssh-copy-id -i ~/.ssh/lab_ed25519.pub <USER>@TARGET_IP
```

For a non-default SSH port:

```bash
ssh-copy-id -i ~/.ssh/lab_ed25519.pub -p <SSH_PORT> <USER>@TARGET_IP
```

Authenticate using the private key:

```bash
ssh -i ~/.ssh/lab_ed25519 <USER>@TARGET_IP
```

The `-i` option specifies the private key to use.

### Method B: Install the Key Manually

**Step 1 — Run on the Attacker machine.**

Display the public key and copy its complete contents:

```bash
cat ~/.ssh/lab_ed25519.pub
```

**Step 2 — Run on the Target**, while logged in as the intended user.

Create the SSH directory and set its permissions:

```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
```

Add the public key to `authorized_keys`. Replace the example key with the complete output from the previous command:

```bash
printf '%s\n' 'ssh-ed25519 AAAA_REPLACE_WITH_THE_COMPLETE_PUBLIC_KEY lab access' >> ~/.ssh/authorized_keys
```

Set the correct permissions and ownership:

```bash
chmod 600 ~/.ssh/authorized_keys
chown -R "$(id -un)":"$(id -gn)" ~/.ssh
```

**Step 3 — Run on the Attacker machine.**

Connect using the private key:

```bash
ssh -i ~/.ssh/lab_ed25519 <USER>@TARGET_IP
```

## 3. Local Port Forwarding with `-L`

Use `-L` when a service is reachable from the SSH server but cannot be accessed directly from the Attacker machine.

**Run on the Attacker machine:**

```bash
ssh -N -L 127.0.0.1:8888:127.0.0.1:8999 <USER>@TARGET_IP
```

The syntax is:

```bash
-L <LOCAL_BIND_ADDRESS>:<LOCAL_PORT>:<DESTINATION_HOST>:<DESTINATION_PORT>
```

In this example:

- `127.0.0.1:8888` is the listening address and port on the Attacker machine.

- `127.0.0.1:8999` is the destination as seen from the Target (SSH server).
   
- `-N` establishes the tunnel without opening an interactive shell.


The two `127.0.0.1` addresses refer to different machines. The first is the Attacker; the second is the Target.

**To access the forwarded service, run on the Attacker machine in another terminal:**

```bash
curl http://127.0.0.1:8888
```

The connection travels through SSH, and the Target connects to its own loopback address on port `8999`.

## 4. Remote Port Forwarding with `-R`

Use `-R` when a service on the Attacker machine needs to be accessed from the remote SSH server.

For example, suppose the Ligolo-ng proxy listens on `127.0.0.1:11601` on the Attacker machine.

**Run on the Attacker machine:**

```bash
ssh -N -R 127.0.0.1:11601:127.0.0.1:11601 <USER>@TARGET_IP
```

The syntax is:

```bash
-R <REMOTE_BIND_ADDRESS>:<REMOTE_PORT>:<DESTINATION_HOST>:<DESTINATION_PORT>
```

In this example:

- The SSH server opens a listener on its own `127.0.0.1:11601`.

- A process on the Target connects to that listener.

- SSH carries the connection back to the Attacker machine's `127.0.0.1:11601`, where the Ligolo-ng proxy is listening.

This is useful when the Ligolo-ng agent can reach the Target's loopback listener but cannot directly reach the Attacker machine.

By default, the remote listener is bound to loopback here, so it is accessible only from the Target itself. Exposing it on other interfaces requires an appropriate bind address and SSH server configuration.

## 5. Combining `-L` and `-R`

You can establish multiple forwards through a single SSH connection. This is useful when you need to expose remote web services locally while also forwarding a connection back to the Attacker machine.

**Run on the Attacker machine:**

```bash
ssh -N \
  -o ExitOnForwardFailure=yes \
  -o ServerAliveInterval=20 \
  -o ServerAliveCountMax=5 \
  -R 127.0.0.1:11601:127.0.0.1:11601 \
  -L 127.0.0.1:5173:127.0.0.1:8091 \
  -L 127.0.0.1:8007:127.0.0.1:8091 \
  app-admin@172.26.10.11
```

What each option does:

- `-R 127.0.0.1:11601:127.0.0.1:11601` allows the remote Ligolo-ng agent to reach the local Ligolo-ng proxy through the Target's loopback listener.

- `-L 127.0.0.1:5173:127.0.0.1:8091` exposes the remote service on local port `5173`.

- `-L 127.0.0.1:8007:127.0.0.1:8091` exposes the same remote service on local port `8007`, satisfying the frontend's hardcoded `localhost:8007` API address in this setup.

- `-N` enables forwarding without opening an interactive shell.

- `ExitOnForwardFailure=yes` terminates SSH if a requested forward cannot be established.

- `ServerAliveInterval` and `ServerAliveCountMax` help detect a dead connection.


**Important:** Both local forwards target the same remote port, `8091`. This is appropriate only if that service is intended to serve both local ports in your setup.

If you also need an interactive shell, remove `-N`.

## 6. Uploading and Downloading Files with SCP

SCP uses SSH for authentication and encryption. The direction of the transfer depends on which side specifies the source file.

### Upload

**Run on the Attacker machine.**

Upload a file to `/tmp` on the Target:

```bash
scp ~/Desktop/Tools/agent <USER>@TARGET_IP:/tmp/agent
```

**Run on the Target** to make the uploaded binary executable:

```bash
chmod +x /tmp/agent
```

Upload using a private key:

```bash
scp -i ~/.ssh/lab_ed25519 ./agent <USER>@TARGET_IP:/tmp/agent
```

Upload through a non-default SSH port. Unlike SSH's lowercase `-p`, SCP uses uppercase `-P`:

```bash
scp -P <SSH_PORT> ./agent <USER>@TARGET_IP:/tmp/agent
```

Upload a directory recursively:

```bash
scp -r ./tools <USER>@TARGET_IP:/tmp/tools
```

### Download

**Run on the Attacker machine.**

Download a file from the Target:

```bash
scp <USER>@TARGET_IP:/tmp/result.txt ./result.txt
```

Download a directory recursively:

```bash
scp -r <USER>@TARGET_IP:/tmp/results ./results
```

## Tunnel Direction at a Glance

|Option|Listener opens on|Destination is reached from|Typical purpose|
|---|---|---|---|
|`-L`|SSH client / Attacker|SSH server / Target|Access a remote or internal service locally|
|`-R`|SSH server / Target|SSH client / Attacker|Let a remote process reach a service on the Attacker|

When troubleshooting, identify three things separately: **where the listening port opens, which machine resolves the destination host, and whether the destination service is actually listening.**


**— 0xchnk**
