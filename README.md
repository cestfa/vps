# VPS Stack

A self-hosted Docker stack for a personal VPS. All services run behind Caddy (reverse proxy + automatic TLS). A single [.env](file:///home/ceest/dev/vps/.env) file configures the entire stack.

## Services Overview

| Service | Subdomain | Purpose |
|---|---|---|
| **AdGuard Home** | `adguard.$DOMAIN` | Network-wide DNS server & ad blocking (with DNS-over-TLS) |
| **AIOMetadata** | `aiometadata.$DOMAIN` | Stremio catalog & metadata enrichment server |
| **AIOStreams** | `aiostreams.$DOMAIN` | Stremio addon aggregator and filter |
| **Beszel** | `beszel.$DOMAIN` | Lightweight resource and container monitoring hub |
| **Dispatcharr** | `dispatcharr.$DOMAIN` | IPTV playlist router and celery-based stream proxy |
| **Filebrowser** | `files.$DOMAIN` | Web file manager to administer configuration files |
| **Ghostfolio** | `ghostfolio.$DOMAIN` | Portfolio tracker and personal wealth manager |
| **Honey** | `$DOMAIN` | Dashboard / start page listing all services |
| **MediaFlow Proxy** | `mediaflow.$DOMAIN` | High-performance edge proxy for Debrid streams |
| **Stirling-PDF** | `pdf.$DOMAIN` | Browser-based PDF manipulation tools |
| **Uptime Kuma** | `uptime.$DOMAIN` | Service uptime monitoring and alerting |
| **Wallos** | `wallos.$DOMAIN` | Recurring subscription tracker |
| **WireGuard VPN** | `vpn.$DOMAIN` | Encrypted tunnel (WG-Easy) to secure all local traffic |

---

## Setup Guide

### 1. Host Preparation

Run these commands on your fresh VPS host to prepare the environment:

```bash
# Install Docker
sudo apt-get update && sudo apt-get install -y curl git
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker $USER
```

**Free port 53 (disable systemd-resolved DNS stub):**

```bash
sudo systemctl stop systemd-resolved
sudo systemctl disable systemd-resolved
echo "nameserver 1.1.1.1" | sudo tee /etc/resolv.conf
```

**Create 2GB swap file (essential for small VPS hosts):**

```bash
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile && sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
echo 'vm.swappiness=10' | sudo tee -a /etc/sysctl.conf && sudo sysctl -p
```

**System memory tuning (required for Redis and databases):**

```bash
echo 'vm.overcommit_memory=1' | sudo tee -a /etc/sysctl.conf && sudo sysctl -p

sudo mkdir -p /etc/systemd/journald.conf.d
printf '[Journal]\nSystemMaxUse=100M\nRuntimeMaxUse=50M\n' | \
  sudo tee /etc/systemd/journald.conf.d/size.conf
sudo systemctl restart systemd-journald
```

**Firewall configuration:**

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp          # SSH
sudo ufw allow 80/tcp          # HTTP
sudo ufw allow 443/tcp         # HTTPS
sudo ufw allow 443/udp         # HTTP/3 (QUIC)
sudo ufw allow 853/tcp         # DNS-over-TLS (AdGuard Private DNS)
sudo ufw allow 51820/udp       # WireGuard VPN tunnel
sudo ufw --force enable
```

**Docker log rotation:**

```bash
sudo mkdir -p /etc/docker
cat <<EOF | sudo tee /etc/docker/daemon.json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
EOF
sudo systemctl restart docker
```

---

### 2. Configure Environment

Clone the repository and copy the environment template:

```bash
sudo git clone https://github.com/cestfa/vps.git /opt/docker
sudo chown -R $USER:$USER /opt/docker
cd /opt/docker
cp .env.example .env
```

Edit your local [.env](file:///home/ceest/dev/vps/.env) and fill in your `DOMAIN` and secrets. You can generate random 32-character hex secrets using:

```bash
openssl rand -hex 32
```

---

### 3. First Launch: AdGuard Home

AdGuard's setup wizard runs on port `3000`. Before starting, open your local [.env](file:///home/ceest/dev/vps/.env) file and temporarily change `ADGUARD_PORT`:

```ini
ADGUARD_PORT=3000
```

Start the stack:

```bash
docker compose up -d
```

Open `https://adguard.$DOMAIN` in your browser and complete the wizard. Make sure to set:
*   **Web interface port:** `80`
*   **DNS server port:** `53`

Once completed, open your [.env](file:///home/ceest/dev/vps/.env) file, restore `ADGUARD_PORT=80`, and apply changes by recreating the Caddy container:

```bash
docker compose up -d caddy
```

---

### 4. WireGuard VPN (WG-Easy) Password Setup

WG-Easy requires a secure bcrypt hash of your password instead of plain text:

1.  Generate the password hash (replace `yourpassword` with your desired password):
    ```bash
    docker run -it ghcr.io/wg-easy/wg-easy wgpw yourpassword
    ```
2.  Copy the output string (e.g. `$2a$12$...`).
3.  Open your local [.env](file:///home/ceest/dev/vps/.env) file and set `WG_EASY_PASSWORD_HASH`.

    > [!IMPORTANT]
    > You must escape all dollar signs (`$`) in your password hash by doubling them (e.g., `$$2a$$12$$...`) in the [.env](file:///home/ceest/dev/vps/.env) file, otherwise Docker Compose will try to interpolate them.

4.  Apply the changes:
    ```bash
    docker compose up -d wg-easy
    ```
5.  Access the VPN dashboard at `https://vpn.$DOMAIN` to generate client configurations and QR codes.

> [!NOTE]
> `WG_DEFAULT_DNS` is set to `10.8.0.1` (the WireGuard gateway address), which routes DNS queries from VPN clients through AdGuard Home running on the VPS. This enables ad blocking and DNS-over-TLS for all connected devices, including when connecting from untrusted networks (e.g. a shared Wi-Fi).

---

### 5. Beszel Agent Integration

After the first stack launch:

1.  Open `https://beszel.$DOMAIN` and create your admin account.
2.  Click **Add System**, choose a system name, and copy the provided `KEY` and `TOKEN` strings.
3.  Open your local [.env](file:///home/ceest/dev/vps/.env) file and paste them into `BESZEL_AGENT_KEY` and `BESZEL_AGENT_TOKEN`.
4.  Restart the agent container to apply settings:
    ```bash
    docker compose up -d beszel-agent
    ```
5.  In the Beszel web panel, set the **Host / IP** address for your system to:
    ```
    /beszel_socket/beszel.sock
    ```

---

### 6. AdGuard DNS-over-TLS (Required for External Devices)

Enables `adguard.$DOMAIN` as a Private DNS provider for mobile devices and remote clients.

In AdGuard → **Settings** → **Encryption settings**:

| Field | Value |
|---|---|
| Enable encryption | ✓ |
| Server name | `adguard.$DOMAIN` |
| Certificate path | `/etc/caddy-certs/caddy/certificates/acme-v02.api.letsencrypt.org-directory/adguard.$DOMAIN/adguard.$DOMAIN.crt` |
| Private key path | `/etc/caddy-certs/caddy/certificates/acme-v02.api.letsencrypt.org-directory/adguard.$DOMAIN/adguard.$DOMAIN.key` |

> [!NOTE]
> This is a **one-time setup**. Because the certificate directory is mapped as a shared volume, AdGuard Home will automatically detect and load updated certificates when Caddy renews them.

---

## Operations & Verification

```bash
# Verify all container statuses
docker compose ps

# Check host and container memory usage
docker stats --no-stream
```
