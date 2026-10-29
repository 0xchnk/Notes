# Pivoting with Ligolo-ng

Ligolo-ng creates a TUN-based tunnel that lets the attack machine route traffic through a reachable target and into an otherwise inaccessible internal network. The proxy runs on the attack machine, while the agent runs on the pivot host.

## Prerequisites

Download the appropriate Ligolo-ng binaries (`proxy` for the Attacker machine and `agent` for the Pivot Host) from the [official GitHub Releases](https://github.com/nicocha30/ligolo-ng/releases). Choose the version matching each machine's operating system and architecture.

For installation instructions and additional options, refer to the [official documentation](https://docs.ligolo.ng/).

In this note, we use `-selfcert` to generate a self-signed TLS certificate automatically when starting the proxy.


## 1. Create the TUN Interface

**Run on the Attacker machine.**

First, remove an old `ligolo` interface if one already exists. This prevents conflicts if you previously created the interface during another session.

```bash
sudo ip link delete ligolo 2>/dev/null
```

Now, create a new TUN interface owned by your current user and bring it up:

```bash
sudo ip tuntap add user $(whoami) mode tun ligolo
sudo ip link set ligolo up
```

The TUN interface acts as the local endpoint through which your attack machine sends packets destined for the internal network. Think of it as the entry point for traffic that will travel through Ligolo-ng.

## 2. Start the Proxy on the Attacker Machine

**Run on the Attacker machine**, inside the directory containing the Ligolo-ng proxy binary.

```bash
sudo ./proxy -selfcert
```

The proxy is the central component running on your attack machine. It accepts incoming agent connections and allows you to establish tunnels through connected pivot hosts.

- `-selfcert` makes the proxy generate and use a self-signed TLS certificate.

- Since the agent command below uses `-ignore-cert`, the agent will accept that certificate without validating its authenticity. This is convenient for an authorized lab, but it does not authenticate the proxy's identity.

**Keep the proxy running** while you perform the remaining steps.

## 3. Start the Agent on the Pivot Host

**Run on the Pivot Host**, inside the directory containing the Ligolo-ng agent binary.

If an old agent process is still running in the background, stop it first:

```bash
pkill agent 2>/dev/null
```

Now, connect the agent to your attack machine:

```bash
./agent -connect OUR_IP:11601 -ignore-cert -retry
```

Replace `OUR_IP` with the IP address of your attack machine that the pivot host can reach. For example, use the address of the attack machine's VPN interface if the pivot host can reach it through that network.

- `-connect OUR_IP:11601` tells the agent which IP address and TCP port to connect to.

- `-ignore-cert` accepts the proxy's self-signed certificate without validating it.

- `-retry` makes the agent attempt to reconnect if the connection drops.


Once the agent connects, return to the **proxy console on the Attacker machine**. Select the agent session and start the tunnel using the Ligolo-ng console commands:

```bash
session
```

Select the appropriate agent session when prompted, then start the tunnel:

```bash
start
```

The exact session selection depends on how many agents are connected. Make sure you select the pivot host you intend to use.

## 4. Route the Internal Network Through Ligolo

**Run on the Attacker machine.**

Before adding a route, identify the internal subnet reachable from the pivot host. For example, if the internal network contains hosts such as `192.168.98.2`, `192.168.98.30`, and `192.168.98.120`, the subnet would normally be `192.168.98.0/24`.

Add or replace the route:

```bash
sudo ip route replace Internal_IP/24 dev ligolo src OwnIPaddress
```

Replace `Internal_IP/24` with the actual internal network in CIDR notation. For the example above, the command would be:

```bash
sudo ip route replace 192.168.98.0/24 dev ligolo src OwnIPaddress
```

This tells your attack machine to send traffic destined for that subnet through the `ligolo` interface instead of using its usual route.

`ip route replace` creates the route if it does not exist or updates the existing route for that destination. Check for conflicts with existing VPN or lab routes before changing them.

>[!Important] Adding the route only directs traffic into the TUN interface. The Ligolo-ng agent must be connected and its tunnel must be running for the traffic to reach the internal network.

## 5. Verify the Route

**Run on the Attacker machine.**

Check which interface Linux will use to reach the internal hosts:

```bash
ip route get 192.168.98.2
ip route get 192.168.98.30
ip route get 192.168.98.120
```

The output should show that traffic to these targets is routed through `ligolo`.

This verifies your local routing configuration, but it does not guarantee that the remote hosts are online or that their firewalls allow the connection.

## 6. Port Forwarding with Ligolo-ng

Port forwarding is useful when you want to access a service through the pivot host without routing an entire internal subnet. For example, you might want to access an internal web server or connect to a service listening on a port that is not directly reachable from your attack machine.

There are two common approaches: **local port forwarding through the Ligolo-ng proxy** and **remote port forwarding**. Choose the one that matches the direction in which you need to expose the service.

### A. Local Port Forwarding

Suppose an internal web server is listening on port `80` at `192.168.98.30`, and you want to access it through port `8080` on your attack machine.

**Run in the Ligolo-ng proxy console on the Attacker machine:**

```bash
listener_add --addr 0.0.0.0:8080 --to 192.168.98.30:80 --tcp
```

This creates a listener on the attack machine's reachable interfaces at TCP port `8080` and forwards incoming connections to `192.168.98.30:80`.

Now, from the attack machine, access the service using:

```bash
curl http://127.0.0.1:8080
```

>[!Note] The example listener binds to `0.0.0.0`, which exposes the listening port on all local IPv4 interfaces. If you only need access from the attack machine itself, bind to `127.0.0.1` instead:

```bash
listener_add --addr 127.0.0.1:8080 --to 192.168.98.30:80 --tcp
```

This is generally the safer choice for local testing.

### B. Remote Port Forwarding

Remote port forwarding is useful when you want a service on the attack machine to be reachable through a listener created on the pivot host.

For example, suppose a service is running on the attack machine at `127.0.0.1:8000`, and you want to expose it through port `9000` on the pivot host.

**Run in the Ligolo-ng proxy console on the Attacker machine:**

```bash
listener_add --addr 0.0.0.0:9000 --to 127.0.0.1:8000 --tcp --remote
```

The `--remote` option creates the listener on the agent side rather than the attack machine side. In this example, connections reaching port `9000` on the pivot host are forwarded to port `8000` on the attack machine.

Use `0.0.0.0` only if the listener needs to be reachable through multiple pivot-host interfaces. Otherwise, bind to the specific interface or address required for your lab.

### C. Verify and Manage Listeners

**Run in the Ligolo-ng proxy console on the Attacker machine.**

To inspect the available listener management commands, use:

```bash
listener_list
```

To remove a listener when you no longer need it, use the listener removal command supported by your installed Ligolo-ng version. Check the proxy console's help output for the exact syntax.

>[!Important] Listener commands and supported options can vary by Ligolo-ng version. If a command is rejected, check the help output for your installed binary rather than assuming that a different version uses the same syntax.

## Traffic Flow

Once everything is configured, the traffic path looks like this:

1. A tool on the Attacker machine sends traffic toward an internal IP address.
2. The route directs that traffic into the local `ligolo` TUN interface.
3. Ligolo-ng forwards the traffic through the connected agent.
4. The Pivot Host sends the traffic toward the internal network from its own network position.
5. Responses travel back through the established tunnel to the Attacker machine.


For port forwarding, the flow depends on which side hosts the listener and which destination you configured. The listener accepts the connection and forwards it to the specified target address and port.

## Common Problems

- **The agent cannot connect:** Confirm that `OUR_IP` is reachable from the Pivot Host and that TCP port `11601` is not blocked. Also, ensure that the proxy is running on the Attacker machine.

- **The agent connects but internal hosts are unreachable:** Confirm that the correct agent session is selected and that the tunnel is running in the proxy console.
   
- **Traffic uses the wrong interface:** Inspect the route using `ip route get <INTERNAL_IP>` and verify that the correct subnet was entered.

- **The connection drops after sleep or a network change:** Restart the proxy or agent as needed. The agent's `-retry` option attempts reconnection while the process remains running.
   
- **A route already exists:** `ip route replace` updates the route, but first verify that changing it will not disrupt another active VPN or lab network.

- **The port-forwarded service is unreachable:** Check the listener configuration, destination IP and port, the relevant firewall rules, and whether the destination service is listening.
   
- **The listener command fails:** Check the help output for your installed Ligolo-ng version because command syntax and supported options can differ.


**— 0xchnk**
