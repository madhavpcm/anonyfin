# Jellyfin + qBittorrent VPN Stack

Gluetun VPN tunnel with WireGuard (via PIA), qBittorrent routed through it, Jellyfin for media streaming.

## Architecture

```
┌──────────────────────────────────────────────────────────┐
│                     Host Machine                         │
│                                                          │
│  ┌──────────────┐     ┌─────────────────────────────┐    │
│  │   Jellyfin   │────▶│     VPN Container (gluetun) │    │
│  │  :8096       │     │  PIA Server --------------------> Anonymous Internet
│  └──────────────┘     └─────────────────────────────┘    │
│                                  │                       │
│                                  │ network_mode: service │
│                                  ▼                       │
│                          ┌─────────────────┐             │
│                          │   qBittorrent   │             │
│                          │   :8080         │             │
│                          └─────────────────┘             │
└──────────────────────────────────────────────────────────┘
```

- **VPN Container**: Gluetun with PIA provider for WireGuard tunnel
- **qBittorrent**: All traffic routed through VPN via fwmark
- **Jellyfin**: Direct access, reaches qBittorrent via internal network

---

## Containers

### 1. VPN Container (`vpn`)

Gluetun VPN tunnel to PIA servers with WireGuard and dynamic port forwarding.

**Login**: N/A - no WebUI

**Configuration**:
```yaml
environment:
  - VPN_SERVICE_PROVIDER=private_internet_access
  - WIREGUARD_IMPLEMENTATION=linux kernel
  - SERVER_REGIONS=<region>                    # e.g., switzerland, netherlands
  - PORT_FORWARDING=on
  - LOCAL_NETWORK=192.168.1.0/24
  - WG_ADDRESS=<your_wg_address>              # e.g., 10.0.0.2/32
  - WG_PRIVATE_KEY=<your_private_key>
  - WG_PUBLIC_KEY=<your_public_key>
  - WG_ENDPOINT_IP=<server_ip>
  - WG_ENDPOINT_PORT=51820
```

**Key Files**:
- `gluetun/` - Gluetun data directory
- `/gluetun/port` - Current forwarded port (updated dynamically)

---

### 2. qBittorrent (`qbittorrent`)

Torrent client with WebUI for remote management. All connections forced through VPN.

**Login**: WebUI at `http://host:8080`

| Field | Value |
|-------|-------|
| Username | `admin` |
| Password | `adminadmin` (pre-configured) |

**Initial Setup**:
1. Spin up the pod once to generate initial config
2. Stop the pod and edit `./qbittorrent/qBittorrent/qBittorrent.conf`:
   ```
   WebUI\Password_PBKDF2="@ByteArray(ARQ77eY1NUZaQsuDHbIMCA==:0WMRkYTUWVT9wVvdDtHAjU9b3b7uB8NR1Gur2hmQCvCDpm39Q+PsJRJPaCU51dEiz+dTzh8qbPsL8WkFljQYFQ==)"
   ```
3. Restart the pod and login with `admin` / `adminadmin`
4. Change to a strong password in WebUI - this will persist on future restarts
5. Port is automatically configured from gluetun's port forwarding

---

### 3. Jellyfin (`jellyfin`)

Media server with WebUI for streaming content.

**Login**: WebUI at `http://host:8096`

| Field | Value |
|-------|-------|
| Username | **Create on first login** |
| Password | **Create on first login** |

**Setup**:
1. Open WebUI in browser
2. Create admin user and password
3. Add media libraries pointing to your download folders
4. Enable DLNA if needed

---

## Getting Passwords & Logs

### VPN Container Logs

```bash
# Check VPN connection status
podman logs vpn 2>&1 | tail -50

# Check port forwarding status
podman exec vpn cat /gluetun/port
```

### All Container Logs

```bash
# Tail all container logs
podman logs -f vpn &
podman logs -f qbittorrent &
podman logs -f jellyfin &
```

---

## Debug Steps

### 1. Verify qBittorrent Traffic Goes Through VPN

```bash
# Check fwmark rules (qBittorrent should use gluetun's fwmark)
sudo nft list ruleset | grep -A5 "fwmark"

# Verify qBittorrent network namespace
podman exec qbittorrent ss -tlnp | grep qbittorrent
```

### 2. Verify VPN is Connected

```bash
# Check VPN status
podman logs vpn 2>&1 | grep -i "connected\|initialized"

# Check for tun device
podman exec vpn ls /dev/tun

# Verify DNS goes through VPN (should show PIA DNS)
podman exec qbittorrent cat /etc/resolv.conf
```

### 3. Test Torrent Connection Through VPN

```bash
# Add a public tracker torrent and monitor connections
# Check peer IP locations - should NOT be your ISP IP

# From inside qBittorrent container:
podman exec qbittorrent curl -s https://am.i.mullvad.net/json
# Should show VPN IP, not your real IP

# Check actual network connections
podman exec qbittorrent ss -tunp | grep qbittorrent
```

### 4. Check Port Forwarding Status

```bash
# Get forwarded port from gluetun
podman exec vpn cat /gluetun/port
```

### 5. Verify Jellyfin Can Access Downloads

```bash
# Check Jellyfin can read the download directory
podman exec jellyfin ls -la /data

# Check permissions
podman exec jellyfin stat /data
```

### 6. Common Connectivity Issues

```bash
# VPN DNS not working - restart VPN container
podman restart vpn

# qBittorrent can't connect - check VPN is up first
podman logs qbittorrent | grep -i error

# Jellyfin shows empty library - check paths match compose file
# Ensure jellyfin user (PUID 1000) can read download folders
sudo chown -R 1000:1000 /path/to/media
```

---

## WireGuard Setup with pia-wg-config

This stack uses WireGuard instead of OpenVPN for better performance. WireGuard configuration is generated using [pia-wg-config](https://github.com/kylegrantlucas/pia-wg-config).

### Installing pia-wg-config

```bash
# Install Go if not installed
sudo apt-get update && sudo apt-get install golang-go

# Install pia-wg-config
go install github.com/kylegrantlucas/pia-wg-config@latest

# Verify installation
pia-wg-config --help
```

### Generating WireGuard Configuration

```bash
# List available regions
pia-wg-config regions

# Generate WireGuard config (outputs to stdout)
pia-wg-config -r <region> PIA_USERNAME PASSWORD

# Generate and save to file
pia-wg-config -r us_california -o wg0.conf your_pia_username your_password
```

The tool generates WireGuard config with:
- Your unique private/public key pair
- PIA server endpoint
- DNS servers
- Persistent keepalive

### Required WireGuard Environment Variables

After generating the config, extract these values for your container environment:

| Variable | Description | Source |
|----------|-------------|--------|
| `WG_ADDRESS` | Your WireGuard client address | From generated config |
| `WG_PRIVATE_KEY` | Your private key | From generated config |
| `WG_PUBLIC_KEY` | Your public key | From generated config |
| `WG_ENDPOINT_IP` | VPN server IP | From generated config (Endpoint) |
| `WG_ENDPOINT_PORT` | VPN server port | From generated config (51820) |
| `PIA_USER` | PIA username | Your PIA account |
| `PIA_PASS` | PIA password | Your PIA account |

---

## Environment Variables

| Variable | Description | Required |
|----------|-------------|----------|
| `PIA_USER` | PIA username | Yes |
| `PIA_PASS` | PIA password | Yes |
| `PIA_LOCATION` | Server region (e.g., switzerland, netherlands) | Yes |
| `LOCAL_NETWORK` | Host network CIDR | Yes |
| `USER_ID` | qBittorrent/Jellyfin user ID | Yes |
| `GROUP_ID` | qBittorrent/Jellyfin group ID | Yes |
| `TIMEZONE` | Timezone | Yes |
| `MEDIA_ROOT` | Path to media files | Yes |
| `WG_ADDRESS` | WireGuard client address | Yes |
| `WG_PRIVATE_KEY` | WireGuard private key | Yes |
| `WG_PUBLIC_KEY` | WireGuard public key | Yes |
| `WG_ENDPOINT_IP` | WireGuard server IP | Yes |
| `WG_ENDPOINT_PORT` | WireGuard server port (default: 51820) | Yes |

---

## Quick Reference

| Service | URL | Credentials |
|---------|-----|-------------|
| Jellyfin | `http://host:8096` | Create account on first login |
| qBittorrent | `http://host:8080` | admin / adminadmin |
| PIA VPN | N/A | WireGuard config via pia-wg-config |

---

## Updating

```bash
# Pull latest images
podman pull lscr.io/linuxserver/jellyfin:latest
podman pull lscr.io/linuxserver/qbittorrent:libtorrentv1
podman pull qmcgaw/gluetun:latest

# Restart containers
podman-compose down
podman-compose up -d
```