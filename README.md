# HomeLab

Docker services for the home lab, managed with Portainer on a two-node Docker
Swarm.

## Current network configuration

The lab uses the `dachshund-octatonic.ts.net` tailnet for private application
access. Tailscale runs directly on both Pis. Each Pi advertises named HTTPS
Services that forward to local application ports. Clients must be connected to
the tailnet and allowed by its access policy. TLS terminates in Tailscale Serve;
the hop from Serve to each local backend uses HTTP.

Docker Swarm continues to use the reserved LAN addresses for cluster traffic
and the `appnet` overlay network. Tailscale does not replace Swarm networking.

| Host | Role | Reserved LAN IP | Tailscale IP |
| --- | --- | --- | --- |
| `raspberrypi4` | Swarm manager; Hermes, Portainer, golink, Pi-hole, Open WebUI | `192.168.86.49` | `100.93.9.106` |
| `raspberrypi3` | Swarm worker; Kasa MCP, Discord MCP | `192.168.86.80` | `100.121.14.84` |

Use DHCP reservations to retain the LAN addresses. Keep the Pi 4 as the only
manager; two nodes do not provide control-plane high availability. Run
`docker stack`, `docker service`, and `docker node` management commands on the
Pi 4. Worker-node errors for these commands on the Pi 3 are expected.

Pi-hole on the Pi 4 provides DNS filtering through two paths: Google Nest Wifi
Pro forwards home-network DNS to `192.168.86.49`, and Tailscale clients use
`100.93.9.106` as their global DNS server. These are two addresses for the same
Pi. The router continues to provide DHCP, and MagicDNS handles tailnet names.
See [Pi-hole DNS](#pi-hole-dns) for configuration and verification steps.

```mermaid
flowchart TB
    client["Mac / Windows / iPhone / iPad<br/>Connected to Tailscale"]

    subgraph tailnet["Tailnet: dachshund-octatonic.ts.net"]
        endpoints["Named HTTPS Services on TCP 443<br/>hermes / portainer / kasa-mcp / discord-mcp / pihole / chat"]

        subgraph pi4["raspberrypi4 - Swarm manager<br/>LAN 192.168.86.49 | Tailscale 100.93.9.106"]
            swarm4["Docker Swarm manager"]
            serve4["Tailscale Serve<br/>hermes + portainer + pihole + chat"]
            hermes["Hermes<br/>host 9119 to container 9119"]
            pihole["Pi-hole - host network<br/>DNS TCP/UDP 53 | admin HTTP 9080"]
            chat["Open WebUI - chat<br/>host 9081 to container 8080"]
            hermesapi["Hermes OpenAI-compatible API<br/>appnet only: tasks.homelab_hermes_agent:8642/v1"]
            portainer["Portainer - separate stack<br/>host 9000 to container 9000"]
            go["golink - separate tailnet device: go<br/>No published host ports"]
            data[("golink_data<br/>Enrollment state + link database")]
        end

        subgraph pi3["raspberrypi3 - Swarm worker<br/>LAN 192.168.86.80 | Tailscale 100.121.14.84"]
            swarm3["Docker Swarm worker"]
            serve3["Tailscale Serve<br/>kasa-mcp + discord-mcp"]
            kasa["Kasa MCP - host network<br/>TCP 8001"]
            discord["Discord MCP - appnet overlay<br/>host 8002 to container 8000"]
        end
    end

    client -->|"HTTPS"| endpoints
    endpoints -->|"hermes / portainer / pihole / chat"| serve4
    endpoints -->|"kasa-mcp / discord-mcp"| serve3
    serve4 -->|"HTTP 127.0.0.1:9119"| hermes
    serve4 -->|"HTTP 127.0.0.1:9000"| portainer
    serve4 -->|"HTTP 127.0.0.1:9080"| pihole
    serve4 -->|"HTTP 127.0.0.1:9081"| chat
    chat -->|"Bearer-authenticated API over appnet"| hermesapi
    hermesapi --- hermes
    lanclients["LAN devices using router DNS"] --> nest["Google Nest Wifi Pro<br/>DHCP + DNS proxy"]
    nest -->|"TCP/UDP 53 to 192.168.86.49"| pihole
    client -->|"Global DNS via Tailscale<br/>TCP/UDP 53 to 100.93.9.106"| pihole
    pihole -->|"DNS"| upstream["Configured upstream DNS resolvers"]
    serve3 -->|"HTTP 127.0.0.1:8001"| kasa
    serve3 -->|"HTTP 127.0.0.1:8002"| discord
    client -->|"http://go/hermes and other shortcuts"| go
    go -->|"Redirect to destination URL"| endpoints
    go --- data
    swarm4 <-->|"Swarm control and overlay traffic over LAN"| swarm3
    kasa -->|"LAN discovery and control"| devices["Kasa devices on trusted LAN"]
```

Golink redirects the client to an application; it does not carry application
traffic after the redirect.

### Service routes and ports

| Tailscale Service | Client URL | Service host | Local HTTP backend |
| --- | --- | --- | --- |
| `svc:hermes` | `https://hermes.dachshund-octatonic.ts.net` | Pi 4 | `127.0.0.1:9119` |
| `svc:portainer` | `https://portainer.dachshund-octatonic.ts.net` | Pi 4 | `127.0.0.1:9000` |
| `svc:pihole` | `https://pihole.dachshund-octatonic.ts.net/admin/` | Pi 4 | `127.0.0.1:9080` |
| `svc:chat` | `https://chat.dachshund-octatonic.ts.net` | Pi 4 | `127.0.0.1:9081` |
| `svc:kasa-mcp` | `https://kasa-mcp.dachshund-octatonic.ts.net/mcp` | Pi 3 | `127.0.0.1:8001` |
| `svc:discord-mcp` | `https://discord-mcp.dachshund-octatonic.ts.net/mcp` | Pi 3 | `127.0.0.1:8002` |

Hermes and Discord use `mode: host` port publishing, so their ports are exposed
only on the node running their task, rather than through the Swarm routing
mesh. Discord's application still listens on container port `8000`; host port
`8002` avoids the existing Portainer port `8000`. Kasa uses the host network
and listens directly on `8001`, with no Docker port mapping.

These backend ports are reachable on their hosts' LAN interfaces too; forwarding
to `127.0.0.1` in Serve does not make a published Docker port loopback-only.
The separate Portainer stack currently publishes ports `8000`, `9000`, and
`9443` through Swarm ingress. Its Tailscale route uses HTTP port `9000`.
Both MCP allowlists include their tailnet names and local backend addresses.

The `appnet` overlay connects Hermes, Discord, golink, and Open WebUI.
Kasa, AirConnect, and Pi-hole use the external Docker `host` network for LAN access.
AirConnect has no placement constraint in the current Compose file.

### Tailnet go links

The `golink` service runs on the Pi 4 and joins Tailscale as a separate device
named `go`. It needs no published host ports. The `golink_data` volume stores
its enrollment state and link database; preserve it and back it up. This is a
regular tailnet device, not an additional `svc:go` Service.

For initial enrollment:

1. Generate an auth key in [Tailscale Keys](https://login.tailscale.com/admin/settings/keys).
   Use `tag:homelab`, disable Reusable and Ephemeral, and enable Pre-approved
   if available. A one-day key expiration is sufficient for enrollment.
2. Set `GOLINK_TS_AUTHKEY` in the Portainer `homelab` stack environment and
   redeploy from the updated Compose file. Never commit the key.
3. Wait for `go` to appear online in Tailscale Machines; approve it if required.
4. Remove the environment variable and redeploy. Enrollment persists in the
   volume, so the key is no longer needed.

With MagicDNS enabled, open `http://go/` from a connected client and create:

| Shortcut | Destination |
| --- | --- |
| `http://go/hermes` | `https://hermes.dachshund-octatonic.ts.net` |
| `http://go/portainer` | `https://portainer.dachshund-octatonic.ts.net` |
| `http://go/pihole` | `https://pihole.dachshund-octatonic.ts.net/admin/` |
| `http://go/chat` | `https://chat.dachshund-octatonic.ts.net` |
| `http://go/kasa` | `https://kasa-mcp.dachshund-octatonic.ts.net/mcp` |
| `http://go/discord` | `https://discord-mcp.dachshund-octatonic.ts.net/mcp` |

Include `http://` if the browser treats a bare `go/hermes` as a search. The full
node name is `go.dachshund-octatonic.ts.net`. Use full HTTPS destination URLs
in MCP clients; MCP endpoints are protocol endpoints, not browser dashboards.

See the [official golink documentation](https://github.com/tailscale/golink).

## 1. Verify the operating system

Run on both Pis:

```bash
dpkg --print-architecture
uname -m
cat /etc/os-release
```

The preferred architecture is `arm64` / `aarch64`. Using Raspberry Pi OS Lite
64-bit on both machines avoids many container-image compatibility problems.

## 2. Set hostnames and reserve addresses

On the Pi 4:

```bash
sudo raspi-config
```

Select **System Options > Hostname**, set it to `raspberrypi4`, and reboot.

On the Pi 3, follow the same process using `raspberrypi3`. Then create DHCP
reservations for both devices in the router.

Verify the configuration on each Pi:

```bash
hostname
hostname -I
```

Verify connectivity from the Pi 4:

```bash
ping -c 3 192.168.86.80
```

Verify connectivity from the Pi 3:

```bash
ping -c 3 192.168.86.49
```

## 3. Update both Pis

```bash
sudo apt-get update
sudo apt-get full-upgrade -y
sudo reboot
```

## 4. Install Docker on both Pis

Raspberry Pi OS 64-bit uses Debian's `arm64` packages. Remove any stale Docker
repository definition, then add Docker's Debian signing key:

```bash
sudo rm -f /etc/apt/sources.list.d/docker.list
sudo apt-get update
sudo apt-get install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/debian/gpg \
  -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

Add Docker's Debian package repository. This supports Debian 13 `trixie` on
`arm64`:

```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" |
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

Install Docker Engine and Compose:

```bash
sudo apt-get update
sudo apt-get install -y \
  docker-ce \
  docker-ce-cli \
  containerd.io \
  docker-buildx-plugin \
  docker-compose-plugin
sudo usermod -aG docker "$USER"
sudo reboot
```

After reconnecting, verify the installation:

```bash
docker version
docker compose version
docker run --rm hello-world
```

See the official [Docker Engine installation instructions for
Debian](https://docs.docker.com/engine/install/debian/) for updates. Docker's
Raspberry Pi OS-specific repository is intended for 32-bit `armhf`; 64-bit
Raspberry Pi OS should use the Debian `arm64` repository.

## 5. Allow Swarm traffic on the trusted LAN

Swarm and Portainer require the following traffic between the nodes:

| Port | Protocol | Purpose |
| --- | --- | --- |
| `2377` | TCP | Swarm control plane |
| `7946` | TCP and UDP | Node discovery |
| `4789` | UDP | Overlay network data |
| `9001` | TCP | Portainer agents |
| `9443` | TCP | Portainer web interface |

Do not expose ports `2377`, `7946`, `4789`, or `9001` through the internet-facing
router. In particular, UDP port `4789` must only be reachable from a trusted
network.

Raspberry Pi OS normally does not enable UFW. Check its status:

```bash
sudo ufw status
```

If UFW is active, allow node traffic between hosts on the trusted LAN:

```bash
sudo ufw allow from 192.168.86.0/24 to any port 2377 proto tcp
sudo ufw allow from 192.168.86.0/24 to any port 7946 proto tcp
sudo ufw allow from 192.168.86.0/24 to any port 7946 proto udp
sudo ufw allow from 192.168.86.0/24 to any port 4789 proto udp
sudo ufw allow from 192.168.86.0/24 to any port 9001 proto tcp
sudo ufw allow from 192.168.86.0/24 to any port 9443 proto tcp
```

See Docker's [Swarm port
requirements](https://docs.docker.com/engine/swarm/swarm-tutorial/) for details.

## 6. Create the Swarm

Run only on `raspberrypi4`, substituting its actual reserved address:

```bash
docker swarm init --advertise-addr 192.168.86.49
```

Docker prints a worker join command containing a token. Treat this token like a
password and do not publish it. To display it again later, run this on the
manager:

```bash
docker swarm join-token worker
```

Run the generated join command on `raspberrypi3`. It will resemble:

```bash
docker swarm join --token YOUR_ACTUAL_WORKER_TOKEN 192.168.86.49:2377
```

Back on the manager, verify that both nodes are ready:

```bash
docker node ls
```

The Pi 4 should show `Leader`; the Pi 3 should have no manager status.

## 7. Label the nodes

Run on `raspberrypi4`:

```bash
docker node update --label-add hardware=pi4 raspberrypi4
docker node update --label-add hardware=pi3 raspberrypi3
docker node update --label-add storage=primary raspberrypi4
```

Verify the labels:

```bash
docker node inspect raspberrypi4 --pretty
docker node inspect raspberrypi3 --pretty
```

## 8. Test scheduling and ingress

On `raspberrypi4`:

```bash
docker service create \
  --name swarm-test \
  --replicas 2 \
  --publish published=8080,target=80 \
  nginx:alpine
docker service ps swarm-test
```

After both replicas are running, open either node address:

- `http://192.168.86.49:8080`
- `http://192.168.86.80:8080`

Remove the test service:

```bash
docker service rm swarm-test
```

## 9. Install Portainer CE

Run only on `raspberrypi4`:

```bash
curl -L \
  https://downloads.portainer.io/ce-lts/portainer-agent-stack.yml \
  -o portainer-agent-stack.yml
less portainer-agent-stack.yml
```

After reviewing the manifest, press `q` and deploy it:

```bash
docker stack deploy \
  --compose-file portainer-agent-stack.yml \
  portainer
```

Check the deployment:

```bash
docker stack services portainer
docker service ps portainer_portainer
docker service ps portainer_agent
```

The Portainer server should report `1/1`, and the global agent service should
report `2/2`. Open the interface at:

```text
https://192.168.86.49:9443
```

The initial self-signed certificate warning is expected. Create the administrator
account promptly. If the initial setup times out, restart the service:

```bash
docker service update --force portainer_portainer
```

The Swarm appears as one Portainer environment. Do not add the Pi 3 as a separate
environment. See Portainer's [Docker Swarm installation
guide](https://docs.portainer.io/2.33-lts/start/install-ce/server/swarm/linux/).

## 10. Deploy a test stack in Portainer

In Portainer, select **Stacks > Add stack > Web editor**, name the stack
`nginx-demo`, and deploy:

```yaml
services:
  web:
    image: nginx:alpine
    ports:
      - target: 80
        published: 8080
        protocol: tcp
        mode: ingress
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.labels.hardware == pi4
      restart_policy:
        condition: on-failure
```

Then open `http://192.168.86.49:8080`.

## 11. Configure Tailscale application access

Install and authenticate Tailscale directly on each Pi:

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo systemctl enable --now tailscaled
sudo tailscale up
```

Join the same tailnet as the client devices. In the Tailscale admin console:

1. Enable MagicDNS and HTTPS certificates under DNS settings.
2. Define `tag:homelab` with `autogroup:admin` as its owner and assign it to both
   Pis. Merge this into the existing access policy rather than replacing it.
3. Define Services named `hermes`, `portainer`, `kasa-mcp`, `discord-mcp`,
   `pihole`, and `chat`, each
   with endpoint `tcp:443`. Allow the intended clients in the tailnet policy.
4. Deploy the current `homelab` Compose definition through Portainer before
   configuring the routes below. This applies host ports and MCP allowlists.

Run on the Pi 4:

```bash
sudo tailscale serve --service=svc:hermes --https=443 http://127.0.0.1:9119
sudo tailscale serve --service=svc:portainer --https=443 http://127.0.0.1:9000
sudo tailscale serve --service=svc:pihole --https=443 http://127.0.0.1:9080
sudo tailscale serve --service=svc:chat --https=443 http://127.0.0.1:9081
```

Run on the Pi 3:

```bash
sudo tailscale serve --service=svc:kasa-mcp --https=443 http://127.0.0.1:8001
sudo tailscale serve --service=svc:discord-mcp --https=443 http://127.0.0.1:8002
```

Approve each advertised host under its Service in the Tailscale admin console.
Service-mode Serve commands persist in the background. Inspect them on either
Pi with:

```bash
sudo tailscale serve status --json
```

From a connected client, open the Hermes and Portainer HTTPS URLs and verify
both MCP health endpoints:

```bash
curl --fail --show-error https://kasa-mcp.dachshund-octatonic.ts.net/healthz
curl --fail --show-error https://discord-mcp.dachshund-octatonic.ts.net/healthz
```

Tailscale provisions the HTTPS certificates for these tailnet names.
Configuration changes belong in `docker-compose.yml` and should be redeployed
through Portainer so future deployments retain them. Tailscale Serve
configuration is stored separately on each host.

See [Tailscale Services](https://tailscale.com/docs/features/tailscale-services)
and the [Linux installation guide](https://tailscale.com/docs/install/linux).

### Verify deployed services

Run on the Pi 4:

```bash
docker stack services homelab
docker service ls --format 'table {{.Name}}\t{{.Ports}}'
```

The `homelab` stack defines AirConnect, Hermes, Discord MCP, Kasa MCP,
golink, Pi-hole, and Open WebUI. Portainer runs in its separate stack.

## Pi-hole and Open WebUI setup

These additions run one replica each on the Pi 4. Before deploying, verify
64-bit ARM support, available memory/storage, and port availability on that Pi:

```bash
dpkg --print-architecture
free -h
df -h /var/lib/docker
sudo ss -lntup '( sport = :53 or sport = :9080 or sport = :9081 )'
```

Use `arm64`. Resolve any existing listeners before deploying; do not disable
system DNS services without identifying what owns port `53`.

In Portainer's `homelab` stack environment, add these values alongside the
existing variables:

| Variable | Value |
| --- | --- |
| `PIHOLE_WEB_PASSWORD` | Strong password for the Pi-hole admin UI |
| `OPEN_WEBUI_SECRET_KEY` | Output of `openssl rand -hex 32`; retain across updates |
| `HERMES_API_SERVER_KEY` | A separate `openssl rand -hex 32` value shared by Hermes and Open WebUI |
| `PIHOLE_DNS_UPSTREAMS` | Optional; defaults to `1.1.1.1;1.0.0.1` |

The three credentials are required by Compose interpolation. Keep them out of
Git and chat. Deploy the updated Compose source through Portainer, then check:

```bash
docker stack services homelab
docker service logs --tail 50 homelab_pihole
docker service logs --tail 50 homelab_open_webui
curl --fail --show-error --output /dev/null http://127.0.0.1:9080/admin/
curl --fail --show-error http://127.0.0.1:9081/health
```

Create `pihole` and `chat` Services with endpoint `tcp:443`, run their Serve
commands in section 11, and approve the Pi 4 for each. Open
`https://pihole.dachshund-octatonic.ts.net/admin/` and
`https://chat.dachshund-octatonic.ts.net`, then add the corresponding go links.

### Pi-hole DNS

Pi-hole uses host networking to serve DNS directly on the Pi 4's LAN and
Tailscale interfaces at TCP/UDP `53`. Its web server uses HTTP port `9080`;
Tailscale handles HTTPS for the admin UI. DHCP and NTP are disabled in Pi-hole;
the router and host continue providing those functions. Its configuration and
DNS databases persist in `pihole_data` mounted at `/etc/pihole`.

Use the address appropriate to the network:

| Setting | Address | Purpose |
| --- | --- | --- |
| Google Nest Wifi Pro custom DNS | `192.168.86.49` | Pi 4's reserved LAN address; reachable from home |
| Tailscale global nameserver | `100.93.9.106` | Pi 4's private Tailscale address; reachable by authorized tailnet clients at home or away |

Confirm the Tailscale address on the Pi 4 with `tailscale ip -4`. DNS settings
take an IP address, without a URL scheme or the admin UI port `9080`.

#### Verify Pi-hole first

Open the [LAN admin UI](http://192.168.86.49:9080/admin/) and sign in with the
password configured in Portainer. Test DNS from a Mac on the home network:

```bash
dig @192.168.86.49 example.com
dig +tcp @192.168.86.49 example.com
```

Both commands should return `NOERROR` and an answer. These direct UDP and TCP
tests succeeded from the Mac on November 8, 2026. They verify LAN resolver
reachability; the checks below verify the router and Tailscale DNS paths.

#### Google Nest Wifi Pro

1. Open **Google Home > Home > Wifi > Network settings > Advanced Networking >
   DHCP IP reservations**. Confirm `raspberrypi4` is reserved as `192.168.86.49`.
2. Return to **Advanced Networking > DNS**, choose **Custom**, and set the
   primary IPv4 DNS server to `192.168.86.49`.
3. Leave secondary IPv4 DNS blank if allowed. If the app requires a value and
   accepts a duplicate, use `192.168.86.49` again. A second Pi-hole would provide
   redundancy; a public secondary can bypass filtering.
4. Save, then reconnect the test client's Wi-Fi or renew its DHCP lease.
5. Temporarily disconnect Tailscale on the Mac to isolate the router path.
   Run `dig example.org` and check Pi-hole's **Query Log** for the lookup.
   Try another uncached name if the router already has the answer cached.

Google Nest Wifi acts as a DNS proxy. Clients can still show the router's LAN
address as their DNS server, and Pi-hole can show the router as the client for
forwarded requests. Keep DHCP enabled on the router and disabled in Pi-hole.

If IPv6 is enabled, check that IPv6 DNS settings also send queries through
Pi-hole. Browser secure DNS and device-level DNS overrides can bypass the
router's resolver. IPv6 can be temporarily disabled under **Advanced Networking
> IPv6** while validating the IPv4 path.

#### Tailscale DNS

1. On the Pi 4, run the following so the DNS server itself does not adopt the
   tailnet DNS configuration:

   ```bash
   sudo tailscale set --accept-dns=false
   ```

2. Connect the Mac to Tailscale and verify both DNS transports before changing
   tailnet settings:

   ```bash
   dig @100.93.9.106 example.com
   dig +tcp @100.93.9.106 example.com
   ```

3. In [Tailscale DNS settings](https://login.tailscale.com/admin/dns), keep
   **MagicDNS** enabled. Select **Add nameserver > Custom** and add
   `100.93.9.106` as a **global nameserver**, without a domain restriction.
4. Remove other global public resolvers if consistent filtering is required,
   then enable **Override DNS servers**. Preserve any intentional split DNS
   settings. If clients use an exit node, enable **Use with exit node** for
   the Pi-hole nameserver.
5. Keep **Use Tailscale DNS settings** enabled on client devices. Any restrictive
   tailnet policy or host firewall must allow those clients to reach the Pi 4
   on both TCP and UDP port `53`.
6. On a phone, turn Wi-Fi off and leave Tailscale connected. Browse new sites
   and confirm the phone's requests appear in Pi-hole's **Query Log**. Also
   confirm existing MagicDNS names and application URLs still resolve.

The Pi-hole admin Service on `443` carries dashboard traffic; DNS goes directly
to the Pi's Tailscale address on port `53`. No subnet route is needed for that
address. The Compose configuration already sets `FTLCONF_dns_listeningMode` to
`ALL` to accept queries on both LAN and Tailscale interfaces.

#### Upstream DNS and recovery

Pi-hole defaults to upstream resolvers `1.1.1.1;1.0.0.1`, configurable through
`PIHOLE_DNS_UPSTREAMS` in Portainer. Public resolvers are appropriate here:
Pi-hole filters queries before forwarding them upstream. Do not point its
upstream DNS back to itself or to the Nest router, which forwards back to
Pi-hole and would create a loop.

Hermes explicitly uses Pi-hole at `192.168.86.49` for external DNS. Docker's
embedded resolver (`127.0.0.11`) still handles Swarm service names. Pi-hole
forwards only `dachshund-octatonic.ts.net` queries to the Pi's local Tailscale
resolver at `100.100.100.100`; public names continue to use the public upstreams.
The forwarding rule is set through `FTLCONF_misc_dnsmasq_lines`. Pi-hole uses
host networking, which makes the device-local Tailscale resolver reachable.
Do not use that resolver as Pi-hole's global upstream, because Tailscale's
global DNS points back to Pi-hole.

Apply DNS configuration changes by redeploying the `homelab` stack through
Portainer using this repository's updated `docker-compose.yml`. Keep the
existing stack environment variables and persistent volumes. The Compose file
is the source of truth for both Hermes's `dns` setting and Pi-hole's forwarding
rule; manage their deployment together through the stack.

Wait for Pi-hole and Hermes to finish starting after the redeploy. Swarm does
not guarantee startup ordering between these services. Pi-hole's restart
briefly interrupts DNS for clients using it, and Hermes can return HTTP 502
through Tailscale Serve until its dashboard is ready. Once DNS is responding,
reconnect Hermes's MCP servers and retry the ChatGPT subscription login.

For verification, open the Pi-hole container's console in Portainer and check
both the tailnet forwarding rule and public DNS:

```bash
nslookup kasa-mcp.dachshund-octatonic.ts.net 192.168.86.49
nslookup auth.openai.com 192.168.86.49
```

Then open the Hermes container's console in Portainer and verify all three
names resolve from that container:

```bash
python - <<'PY'
import socket

for host in (
    "auth.openai.com",
    "kasa-mcp.dachshund-octatonic.ts.net",
    "discord-mcp.dachshund-octatonic.ts.net",
):
    print(host, socket.gethostbyname_ex(host))
PY
```

`Temporary failure in name resolution` inside Hermes indicates a container DNS
problem even if the Pi host resolves the same name. A Pi-hole `NXDOMAIN` for a
tailnet name indicates the tailnet forwarding path needs checking. Disabling
ad blocking does not repair either resolver configuration issue.

Keep DNS port `53` reachable only from trusted networks; do not forward it from
the internet-facing router. This setup has one filtering DNS server, so keep
the Pi 4 running. For recovery, restore Google Home DNS to **Automatic** and
restore the previous Tailscale nameservers/override setting. An affected client
can temporarily disable **Use Tailscale DNS settings**; at home, it will still
depend on the router's DNS configuration.

See [Google DNS settings](https://support.google.com/googlehome/answer/6274141?hl=en),
[Google DHCP reservations](https://support.google.com/googlehome/answer/6274660?hl=en),
[Tailscale's Pi-hole guide](https://tailscale.com/docs/solutions/block-ads-all-devices-anywhere-using-raspberry-pi),
[Tailscale DNS](https://tailscale.com/docs/reference/dns-in-tailscale),
[Pi-hole Docker configuration](https://docs.pi-hole.net/docker/configuration/),
and [FTL settings](https://docs.pi-hole.net/ftldns/configfile/).

### Open WebUI with Hermes

Open WebUI publishes host port `9081` to container port `8080` on the Pi 4.
Its `main-slim` image omits bundled models for a smaller installation; no
Ollama or additional inference service is installed. Chat history, users, and
settings persist in `open_webui_data` mounted at `/app/backend/data`.

The Compose definition enables Hermes's API on container port `8642` bound to
`0.0.0.0`. That API port is not published to the host. Open WebUI reaches
`http://tasks.homelab_hermes_agent:8642/v1` over `appnet`, authenticating with the same
`HERMES_API_SERVER_KEY` supplied to Hermes. This is separate from Hermes's
dashboard on `9119`; no browser-side connection or API CORS setting is needed.

The `tasks.homelab_hermes_agent` DNS name resolves directly to Hermes task
addresses. It bypasses the service VIP, which failed TCP connectivity checks
while direct task connections succeeded. This restores a usable path without
repairing the underlying VIP routing issue. Keep the DNS name rather than a
container IP, because task addresses change on replacement. The name assumes
the Portainer stack is named `homelab`.

Open `https://chat.dachshund-octatonic.ts.net` and create the first account,
which becomes the administrator. In Admin Settings, verify the OpenAI-compatible
connection URL is `http://tasks.homelab_hermes_agent:8642/v1`, with the Hermes API key. Select
`hermes-agent` and test a chat. Hermes retains its existing model-provider and
tool configuration. Disable further signups in Admin Settings if only your
account should use the service. Chats submitted here can invoke Hermes tools
with the gateway's existing permissions.

Connection settings seed the initial database and can later be overridden by
saved Admin Settings. If changing environment values after first launch, check
the saved connection settings too. The slim image's embedding, transcription,
and document features may require separate providers; basic Hermes chat does not.

See the [Hermes integration guide](https://hermes-agent.nousresearch.com/docs/user-guide/messaging/open-webui/)
and [Open WebUI quick start](https://docs.openwebui.com/getting-started/quick-start/).

## Storage and migration notes

Swarm does not automatically copy a named volume or bind-mounted data when it
reschedules a service. Keep stateful services on the node containing their data
by combining an absolute bind mount with a placement constraint:

```yaml
services:
  app:
    image: some/image:latest
    volumes:
      - /srv/app-data:/config
    deploy:
      placement:
        constraints:
          - node.labels.storage == primary
```

Create and back up the directory on the Pi 4:

```bash
sudo mkdir -p /srv/app-data
```

Do not run multiple replicas of a database unless the database itself is
configured for replication. Multiple replicas sharing an ordinary directory can
corrupt the data.

The repository's `docker-compose.yml` is deployed as the `homelab` Swarm stack
through Portainer. Placement constraints keep Hermes, golink, Pi-hole, and Open WebUI on the Pi 4,
and both MCP servers on the Pi 3. Keep these deployment details in mind:

- `airconnect` and `kasa_mcp_server` use host networking for LAN discovery.
- `container_name`, standalone `restart`, and some Compose security options
  are not supported by Swarm stacks; `deploy.restart_policy` controls restarts.
- Supply deployment variables through Portainer or suitable secrets/configs.
  Do not commit secret values.
- Verify image support for `linux/arm64` before changing images.

Migrate one non-critical service at a time and confirm its networking, storage,
health check, and restart behavior before moving the next service.

## Discord MCP server

`discord_mcp_server` runs one replica on `raspberrypi3` using the shared
`appnet` overlay network. Container port `8000` is published as host port `8002`
on the Pi 3 for Tailscale Serve. Configure MCP clients with
`https://discord-mcp.dachshund-octatonic.ts.net/mcp`.

Supply `DISCORD_MCP_BOT_TOKEN` and `DISCORD_MCP_GUILD_ID` through the deployment
environment (or Portainer stack variables). Optional
`DISCORD_MCP_ALLOWED_CHANNEL_IDS` defaults to `*` for all message-capable
channels in the configured guild; set comma-separated channel IDs to restrict
access. Optional Sentry settings are `DISCORD_MCP_SENTRY_DSN`,
`DISCORD_MCP_SENTRY_ENVIRONMENT` (default `production`), and
`DISCORD_MCP_SENTRY_TRACES_SAMPLE_RATE` (default `1.0`). Do not commit tokens.

The service exposes `/healthz` for its container health check and through its
Tailscale HTTPS endpoint. Redeploy the stack through Portainer after changing
its deployment variables or Compose settings.
