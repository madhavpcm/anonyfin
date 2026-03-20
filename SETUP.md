# Jellyfin + qBittorrent VPN Stack

WireGuard VPN tunnel with qBittorrent routed through it, Jellyfin for media streaming.

## Architecture

```
┌──────────────────────────────────────────────────────────┐
│                     Host Machine                         │
│                                                          │
│  ┌──────────────┐     ┌─────────────────────────────┐    │
│  │   Jellyfin   │────▶│     VPN Container (wg0)     │    │
│  │  :8096       │     │  WireGuard PIA Server --------------> Anonymous Internet
│  └──────────────┘     └─────────────────────────────┘    │
│                                  │                       │
│                                  │ network_mode: service │
│                                  ▼                       │
│                          ┌─────────────────┐             │
│                          │   qBittorrent   │             │
│                          │   :8080         │             x
│                          └─────────────────┘             │
└────────────────────────────────────────────────────────x─┘
```

- **VPN Container**: Creates WireGuard tunnel to PIA
- **qBittorrent**: All traffic routed through VPN via fwmark
- **Jellyfin**: Direct access, reaches qBittorrent via internal network

---

## Containers

### 1. VPN Container (`pia-wireguard`)

WireGuard VPN tunnel to PIA servers. All VPN magic happens here.

**Login**: N/A - no WebUI

**Configuration**:
```yaml
environment:
  - PIA_USER=<your_pia_username>
  - PIA_PASS=<your_pia_password>
  - PIA_SERVER=<server_location>          # e.g., switzerland, netherlands
  - VPN_PORT_FORWARDING=enabled
  - LOCAL_NETWORK=192.168.1.0/24
```

**Key Files**:
- `pia/private_key` - WireGuard private key
- `pia-shared/` - Port forwarding scripts and state

---

### 2. qBittorrent (`qbittorrent`)

Torrent client with WebUI for remote management. All connections forced through VPN.

**Login**: WebUI at `http://host:8080`

| Field | Value |
|-------|-------|
| Username | `admin` |
| Password | **Get from container logs** (see below) |

**Initial Setup**:
1. Change default password immediately
2. Set **Listening Port** to `51697` (PIA forwarded port) - Settings → Connection → Listening Port
3. Enable WireGuard port forwarding integration (Settings → Advanced → Network Interface → `wg0`)
4. Verify connection: Settings → Connection → "Bind to IP address" should be `0.0.0.0`

> **Note**: PIA forwards port 51697. This port must match qBittorrent's listening port for incoming peers. See [LinuxServer qBittorrent docs](https://docs.linuxserver.io/images/docker-qbittorrENT).

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

### qBittorrent Password

```bash
# Get the auto-generated password from logs
podman logs qbittorrent 2>&1 | grep -i password
```


### VPN Container Logs

```bash
# Check VPN connection status
podman logs pia-wireguard 2>&1 | tail -50

# Check port forwarding status
podman exec pia-wireguard cat /pia-shared/port_forwarding_status.json
```

### All Container Logs

```bash
# Tail all container logs
podman logs -f pia-wireguard &
podman logs -f qbittorrent &
podman logs -f jellyfin &
```

---

## Debug Steps

### 1. Verify qBittorrent Traffic Goes Through VPN

Check if qBittorrent processes are routing through WireGuard:

```bash
# Check fwmark rules (qBittorrent should use 0xca6c)
sudo nft list ruleset | grep -A5 "fwmark"

# Check VPN routing table
ip route show table 51820

# Verify qBittorrent network namespace
podman exec qbittorrent ss -tlnp | grep qbittorrent
```

### 2. Verify VPN is Connected

```bash
# Check WireGuard interface
ip link show wg0
ip addr show wg0

# Check if traffic flows through wg0
watch -n1 'sudo ip -s link show wg0'

# Verify no DNS leaks (should show PIA DNS)
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
# Get forwarded port from PIA
podman exec pia-wireguard cat /pia-shared/port_forwarding_status.json

# Verify port is open externally (optional)
curl "https://check-host.net/check-port?host=YOUR_PIA_IP&port=YOUR_FORWARDED_PORT"
```

### 5. Verify Jellyfin Can Access Downloads

```bash
# Check Jellyfin can read the download directory
podman exec jellyfin ls -la /media/downloads

# Check permissions
podman exec jellyfin stat /media/downloads
```

### 6. Common Connectivity Issues

```bash
# VPN DNS not working - restart VPN container
podman restart pia-wireguard

# qBittorrent can't connect - check VPN is up first
podman logs qbittorrent | grep -i error

# Jellyfin shows empty library - check paths match compose file
# Ensure jellyfin user (PUID 1000) can read download folders
sudo chown -R 1000:1000 /path/to/media
```

---

## Environment Variables

| Variable | Description | Required |
|----------|-------------|----------|
| `PIA_USER` | PIA username | Yes |
| `PIA_PASS` | PIA password | Yes |
| `PIA_SERVER` | Server region (e.g., switzerland, netherlands) | Yes |
| `VPN_PORT_FORWARDING` | Enable PIA port forwarding | Yes |
| `LOCAL_NETWORK` | Host network CIDR | Yes |
| `WEBUI_PORT` | qBittorrent WebUI port | Yes |
| `PUID/PGID` | Jellyfin user ID | Optional |
| `TZ` | Timezone | Optional |

---

## Quick Reference

| Service | URL | Credentials |
|---------|-----|-------------|
| Jellyfin | `http://host:8096` | Create account on first login |
| qBittorrent | `http://host:8080` | admin / [from logs] |
| PIA VPN | N/A | user + pass + location in env |

---

## Updating

```bash
# Pull latest images
podman pull lscr.io/linuxserver/jellyfin:latest
podman pull lscr.io/linuxserver/qbittorrent:libtorrentv1
podman pull ghcr.io/thrnz/docker-wireguard-pia:latest

# Restart containers
podman-compose down
podman-compose up -d
```
