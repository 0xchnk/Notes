# SSH Access, Tunneling, and File Transfers

SSH provides encrypted remote shell access, but it can also forward TCP connections and transfer files. This note covers normal SSH access, public-key authentication, local and remote port forwarding, combined tunnels, and SCP.

> [!WARNING]
> Use these commands only on systems and networks where you have explicit authorization. Forwarded ports can expose services beyond their intended network boundary if bound carelessly.

## 1. Normal SSH Login

Connect using a username and password:

```bash
ssh <USER>@TARGET_IP
```

Connect to a non-default SSH port:

```bash
ssh -p <SSH_PORT> <USER>@TARGET_IP
```

Useful first-connection options:

```bash
ssh -o ServerAliveInterval=30 -o ServerAliveCountMax=3 <USER>@TARGET_IP
```

- `ServerAliveInterval=30` sends a keepalive message every 30 seconds while no data is received.
- `ServerAliveCountMax=3` closes the connection after three unanswered keepalive messages.

These options help detect dead connections and are especially useful when working through VPNs, pivots, or unstable lab networks.

## 2. SSH Public-Key Authentication

### Generate a Key Pair on the Attack Machine

Generate a modern Ed25519 key pair:

```bash
ssh-keygen -t ed25519 -a 100 -f ~/.ssh/lab_ed25519 -C "lab access"
```

This creates:

```text
~/.ssh/lab_ed25519      # Private key — do not share it
~/.ssh/lab_ed25519.pub  # Public key — place this on the target
```

Protect the private key:

```bash
chmod 600 ~/.ssh/lab_ed25519
```

The private key stays on the attack machine. The public key is installed in the intended user's `~/.ssh/authorized_keys` file on the target.

### Method A: Install the Key with `ssh-copy-id`

If password-based SSH access already works:

```bash
ssh-copy-id -i ~/.ssh/lab_ed25519.pub <USER>@TARGET_IP
```

For a non-default SSH port:

```bash
ssh-copy-id -i ~/.ssh/lab_ed25519.pub -p <SSH_PORT> <USER>@TARGET_IP
```

Authenticate using the private key:

```bash
ssh -i lab_ed25519 <USER>@TARGET_IP
```

The relative path in this command assumes that `lab_ed25519` is in the current directory. Otherwise, use its full path, such as `~/.ssh/lab_ed25519`.

### Method B: Install the Key Manually

Display the public key on the attack machine:

```bash
cat ~/.ssh/lab_ed25519.pub
```

On the target, while logged in as the intended user:

```bash
mkdir -p ~/.ssh

chmod 700 ~/.ssh

printf '%s\n' 'ssh-ed25519 AAAA_REPLACE_WITH_THE_COMPLETE_PUBLIC_KEY lab access' >> ~/.ssh/authorized_keys

chmod 600 ~/.ssh/authorized_keys

chown -R "$(id -un)":"$(id -gn)" ~/.ssh
```

Then connect from the attack machine:

```bash
ssh -i lab_ed25519 <USER>@TARGET_IP
```

> [!WARNING]
> Add only the public key ending in `.pub` to `authorized_keys`. Never copy the private key onto the target.

## 3. Local Port Forwarding with `-L`

Use `-L` when a service is reachable from the SSH server but is not directly reachable from the attack machine.

```bash
ssh -N -L 127.0.0.1:8888:127.0.0.1:8999 <USER>@TARGET_IP
```

The syntax is:

```text
-L <LOCAL_BIND_ADDRESS>:<LOCAL_PORT>:<DESTINATION_HOST>:<DESTINATION_PORT>
```

In the example:

- SSH opens `127.0.0.1:8888` on the attack machine.
- A connection to that local port travels through SSH.
- The SSH server then connects to `127.0.0.1:8999` from its own network perspective.

The two `127.0.0.1` addresses refer to different machines: the first is the attack machine, while the second is the SSH server.

## 4. Remote Port Forwarding with `-R`

Use `-R` when the SSH server or compromised machine needs to reach a service on the attack machine, especially when direct target-to-attacker connections are blocked.

If the Ligolo proxy listens on `127.0.0.1:11601` on the attack machine:

```bash
ssh -N -R 127.0.0.1:11601:127.0.0.1:11601 <USER>@TARGET_IP
```

The syntax is:

```text
-R <REMOTE_BIND_ADDRESS>:<REMOTE_PORT>:<DESTINATION_HOST>:<DESTINATION_PORT>
```

In the example:

- SSH opens `127.0.0.1:11601` on the SSH server.
- A program on that server connects to the remote listener.
- SSH carries the connection back to the client and reaches `127.0.0.1:11601` on the attack machine.

This is useful when the remote Ligolo agent can reach only the SSH server's loopback listener, while SSH provides the path back to the local proxy.

## 5. Combining `-L` and `-R`

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

- `-R 11601` allows the remote Ligolo agent to reach the local Ligolo proxy.
- `-L 5173` exposes the remote HotHost frontend on the attack machine.
- `-L 8007` satisfies the frontend's hardcoded `localhost:8007` API address.
- `-N` enables forwarding without opening an interactive shell.
- `ExitOnForwardFailure=yes` terminates SSH if a requested listener cannot be created.
- `ServerAliveInterval` and `ServerAliveCountMax` help detect a dead tunnel.

An SSH tunnel also works with an interactive shell. Omit `-N` if both a shell and the forwards are needed.

## 6. Uploading and Downloading Files with SCP

SCP uses SSH for authentication and encryption. Its source and destination syntax determines the direction of the transfer.

### Upload

Upload one file to `/tmp`:

```bash
scp ~/Desktop/Tools/agent <USER>@TARGET_IP:/tmp/agent
```

Make it executable on the target:

```bash
chmod +x /tmp/agent
```

Upload with a private key:

```bash
scp -i ~/.ssh/lab_ed25519 ./agent <USER>@TARGET_IP:/tmp/agent
```

Upload through a non-default SSH port. SCP uses uppercase `-P` for the port:

```bash
scp -P <SSH_PORT> ./agent <USER>@TARGET_IP:/tmp/agent
```

Upload a directory recursively:

```bash
scp -r ./tools <USER>@TARGET_IP:/tmp/tools
```

### Download

Download a file from the target:

```bash
scp <USER>@TARGET_IP:/tmp/result.txt ./result.txt
```

Download a directory:

```bash
scp -r <USER>@TARGET_IP:/tmp/results ./results
```

## Tunnel Direction at a Glance

| Option | Listener opens on | Destination is reached from | Typical purpose |
| --- | --- | --- | --- |
| `-L` | SSH client / attack machine | SSH server | Reach a remote or internal service locally |
| `-R` | SSH server / remote machine | SSH client | Let a remote process reach a service on the attack machine |

When troubleshooting, identify three things separately: where the listening port opens, which machine resolves the destination host, and whether that destination service is actually listening.
