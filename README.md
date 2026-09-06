# HomeLab

Docker services for the home lab, managed with Portainer on a two-node Docker
Swarm.

## Proposed topology

| Host | Role | IP address |
| --- | --- | --- |
| `raspberrypi4` | Swarm manager and primary workload node | `192.168.86.49` |
| `raspberrypi3` | Swarm worker for lightweight workloads | `192.168.86.80` |

Use DHCP reservations in the router so both hosts retain these addresses.

Two nodes do not provide control-plane high availability. Keep the Pi 4 as the
only manager. A resilient Swarm manager quorum requires three manager nodes.

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

## 11. Configure a Let's Encrypt wildcard certificate

The public DNS zone for `tutkowski.com` is hosted by Amazon Route 53. Use
Let's Encrypt's DNS-01 challenge with Certbot's Route 53 plugin to obtain a
wildcard certificate. DNS-01 does not require the services themselves to be
publicly reachable, but the validation TXT record must be visible in public
DNS.

Request both `tutkowski.com` and `*.tutkowski.com`. The wildcard alone does not
cover the apex domain, and it covers only one subdomain label. For example, it
covers `portainer.tutkowski.com`, but not
`app.portainer.tutkowski.com`.

Run the following steps on the node where `nginx-proxy` terminates TLS. These
instructions use the repository's standalone `docker-compose.yml`; if the
proxy is later moved into the Swarm, constrain Certbot and Nginx to the node
that contains the certificate volume.

### Create a restricted Route 53 IAM identity

In AWS IAM, create a dedicated policy for Certbot. Replace
`YOUR_HOSTED_ZONE_ID` with the public Route 53 hosted-zone ID for
`tutkowski.com`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "route53:ListHostedZones",
        "route53:GetChange"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": "route53:ChangeResourceRecordSets",
      "Resource": "arn:aws:route53:::hostedzone/YOUR_HOSTED_ZONE_ID",
      "Condition": {
        "ForAllValues:StringEquals": {
          "route53:ChangeResourceRecordSetsNormalizedRecordNames": [
            "_acme-challenge.tutkowski.com"
          ],
          "route53:ChangeResourceRecordSetsRecordTypes": [
            "TXT"
          ],
          "route53:ChangeResourceRecordSetsActions": [
            "CREATE",
            "UPSERT",
            "DELETE"
          ]
        }
      }
    }
  ]
}
```

Attach the policy to a dedicated IAM user and create an access key.

On the Pi 4, store the access key outside this repository:

```bash
sudo install -d -m 700 /opt/certbot/aws
sudoedit /opt/certbot/aws/credentials
sudo chmod 600 /opt/certbot/aws/credentials
```

Enter the credentials using the standard AWS format:

```ini
[default]
aws_access_key_id = YOUR_ACCESS_KEY_ID
aws_secret_access_key = YOUR_SECRET_ACCESS_KEY
```

Treat this file like a password. Never commit it or place it in a Docker image.

### Add Certbot to Docker Compose

Add port 443 and a read-only certificate volume to `nginx-proxy`:

```yaml
services:
  nginx-proxy:
    # Keep the existing settings.
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - letsencrypt:/etc/letsencrypt:ro
```

Add a Certbot service at the same level as `nginx-proxy`:

```yaml
  certbot:
    image: certbot/dns-route53:latest
    profiles:
      - tools
    volumes:
      - letsencrypt:/etc/letsencrypt
      - /opt/certbot/aws:/root/.aws:ro
```

Add the shared named volume at the bottom of the file, merging it with any
existing `volumes` section:

```yaml
volumes:
  letsencrypt:
```

Do not add the Nginx TLS directives yet. Nginx will fail to start if its
configuration references certificate files that have not been created.

### Issue the initial certificate

From the directory containing `docker-compose.yml`, run:

```bash
docker compose run --rm certbot certonly \
  --dns-route53 \
  --non-interactive \
  --agree-tos \
  --email YOUR_EMAIL_ADDRESS \
  --cert-name tutkowski.com \
  -d tutkowski.com \
  -d '*.tutkowski.com'
```

Certbot temporarily creates `_acme-challenge.tutkowski.com` TXT records in
Route 53, waits for validation, and removes them afterward. The resulting files
are stored in the `letsencrypt` volume under
`/etc/letsencrypt/live/tutkowski.com/`.

### Enable TLS in Nginx

After the certificate exists, add the following settings to each HTTPS virtual
host in `nginx.conf`:

```nginx
listen 443 ssl;
ssl_certificate /etc/letsencrypt/live/tutkowski.com/fullchain.pem;
ssl_certificate_key /etc/letsencrypt/live/tutkowski.com/privkey.pem;
ssl_protocols TLSv1.2 TLSv1.3;
```

Use a separate port 80 server block to redirect HTTP to HTTPS while preserving
the default `/healthz` endpoint:

```nginx
server {
    listen 80;
    server_name portainer.tutkowski.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name portainer.tutkowski.com;

    ssl_certificate /etc/letsencrypt/live/tutkowski.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/tutkowski.com/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;

    location / {
        proxy_pass http://host.docker.internal:9000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-Proto https;
    }
}
```

Repeat that pattern for each proxied hostname. Test the generated Nginx
configuration before recreating the proxy:

```bash
docker compose run --rm --no-deps nginx-proxy nginx -t
docker compose up -d nginx-proxy
```

### Test and automate renewal

Test Certbot's renewal path against Let's Encrypt's staging environment:

```bash
docker compose run --rm certbot renew --dry-run
```

Then edit root's crontab:

```bash
sudo crontab -e
```

Add the following entry, replacing `/path/to/HomeLab` with the absolute path on
the Docker host:

```cron
17 3,15 * * * cd /path/to/HomeLab && /usr/bin/docker compose run --rm certbot renew --quiet && /usr/bin/docker compose exec -T nginx-proxy nginx -s reload
```

The job checks twice daily. Certbot renews only when the certificate is near
expiry, and Nginx reloads the certificate files without stopping the proxy.

The wildcard certificate secures hostnames; it does not create DNS records for
them. Each service still needs an appropriate Route 53 A, AAAA, or CNAME record
unless a separate wildcard DNS record is used.

For background and current requirements, see the official
[Let's Encrypt challenge documentation](https://letsencrypt.org/docs/challenge-types/),
[Certbot Route 53 plugin documentation](https://certbot-dns-route53.readthedocs.io/en/stable/),
and [AWS Route 53 record-level IAM documentation](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resource-record-sets-permissions.html).

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

The repository's current `docker-compose.yml` is designed for standalone Docker
and should not be deployed to Swarm unchanged:

- `airconnect` and `kasa_mcp_server` use host networking for LAN discovery and
  should be pinned to an appropriate node or remain standalone.
- `container_name` and some modern Compose options do not translate directly to
  Swarm stacks.
- Variables currently supplied from `.env` must be entered in Portainer or
  converted to Swarm secrets/configs. Do not commit secret values.
- Verify that every image supports `linux/arm64` before allowing Swarm to place it
  on either Pi.

Migrate one non-critical service at a time and confirm its networking, storage,
health check, and restart behavior before moving the next service.
