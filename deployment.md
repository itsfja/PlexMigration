[plex_lxc_htpc_deployment_notes.md](https://github.com/user-attachments/files/34163118/plex_lxc_htpc_deployment_notes.md)
# Plex Media Server (LXC) & Plex HTPC Cross-VLAN Deployment Runbook

## 1. Architectural Overview

* **Host Environment:** Proxmox VE 8.x (`pve1` / Intel NUC cluster).
* **Container:** Debian 13 LXC (Unprivileged, Container ID `201`, Hostname `plex-lxc`).
* **Compute / Transcoder:** Intel QuickSync Video via 10th Gen CometLake-U UHD Graphics (`8086:9bca:8086:2081@0000:00:02.0`).
* **Hardware Device Node:** `/dev/dri/renderD128` (gid 107/44 passthrough).
* **Network Segmentation:**
  * **Server Subnet (VLAN 3 / IoT / Services):** `192.168.3.0/24` (Plex LXC IP: `192.168.3.57`).
  * **Client Subnet (VLAN 10 / Gaming LAN):** `192.168.10.0/24` (GamesPC IP: `192.168.10.165`).
  * **Router / Firewall:** UniFi Gateway handling inter-VLAN routing.
* **Client Software:** Plex HTPC for Windows (mpv-based embedded player) driving a 3440x1440p ultrawide display.

---

## 2. Identified Failure Modes & Root Causes

### A. Loopback SSH Tunnel Poisoning
* **Symptom:** Plex HTPC logged persistent `Server's certificate does not match the URL.-200` errors targeting `https://127-0-0-1.<hash>.plex.direct:32400`.
* **Root Cause:** Initial setup and claiming of the Plex server occurred via an SSH local port forward (`localhost:32400`). Plex registered `127.0.0.1` with `plex.tv` as an active local address. When Plex HTPC queried `plex.tv`, it attempted TLS handshakes to `127.0.0.1` on the local Windows PC instead of the remote container, aborting the connection.

### B. DNS Rebinding Protections Across VLANs
* **Symptom:** Client failed to establish direct SSL sessions across subnets, triggering fallback loops and `notConnected` statuses.
* **Root Cause:** Plex utilizes dynamic subdomains (`*.plex.direct`) mapped to private IP addresses. Upstream recursive DNS resolvers and UniFi firewall rules prevent public domains from resolving to RFC 1918 private subnets (DNS Rebinding Attack mitigation).

### C. Client Player Abortion (Audio Passthrough)
* **Symptom:** "An unexpected playback problem occurred" on initial playback launch across all media titles.
* **Root Cause:** Plex HTPC defaults to attempting hardware bitstream audio passthrough over HDMI/DisplayPort. When the client output device does not support lossless codecs (TrueHD, DTS-HD MA, EAC3), mpv fails initialization and terminates playback immediately.

### D. Global Transcode Downsampling
* **Symptom:** Video streams defaulting universally to 480p SD transcodes regardless of local bandwidth.
* **Root Cause:** Setting playback quality manually in the client player HUD during a single transcode test sets a global persistent ceiling across all subsequent playback sessions.

---

## 3. Server Configuration

### File: `/var/lib/plexmediaserver/Library/Application Support/Plex Media Server/Preferences.xml`

Ensure the `<Preferences ... />` node contains the following exact attributes:

```xml
customConnections="http://192.168.3.57:32400"
SecureConnections="0"
lanNetworks="192.168.3.0/24,192.168.10.0/24"
allowedNetworks="192.168.3.0/24,192.168.10.0/24"
TranscoderTempDirectory="/dev/shm"
HardwareDevicePath="8086:9bca:8086:2081@0000:00:02.0"
```

* **`customConnections`**: Forces Plex to publish the exact routable container LAN IP to `plex.tv`, overriding loopback addresses.
* **`SecureConnections="0"`**: Disables mandatory TLS/SSL enforcement, preventing `*.plex.direct` certificate mismatches across subnets.
* **`lanNetworks`**: Treats VLAN 10 (`192.168.10.0/24`) as a local LAN, bypassing remote WAN streaming bandwidth caps.
* **`allowedNetworks`**: Bypasses token authentication barriers for local subnet management.
* **`TranscoderTempDirectory="/dev/shm"`**: Directs temporary transcoder chunks to RAM tmpfs, reducing NVMe wear.

### Service Commands (LXC Console)
```bash
# Restart Plex daemon
systemctl restart plexmediaserver

# Clear stale certificate cache
systemctl stop plexmediaserver
rm -rf "/var/lib/plexmediaserver/Library/Application Support/Plex Media Server/Cache/certificate"*
systemctl start plexmediaserver

# Format and inspect Preferences.xml cleanly
apt install -y xmlstarlet
xmlstarlet fo -s 2 "/var/lib/plexmediaserver/Library/Application Support/Plex Media Server/Preferences.xml"
```

---

## 4. Client Configuration (Plex HTPC on Windows)

### Audio Settings
1. Navigate to **Settings -> Audio**.
2. **Audio Output Device:** Set to current audio endpoint (e.g. HDMI / TV / Monitor).
3. **Channels:** Set to `Stereo` (or discrete layout supported by hardware without external AVR).
4. **Audio Passthrough:** `Disabled` (ensures server downmixes or transcodes audio to avoid player aborts).

### Network & Advanced Settings
1. Navigate to **Settings -> Advanced**.
2. **Allow Insecure Connections:** Set to `Always`.
3. **Prefer Insecure Connections:** Set to `Always` (or `On same network`).

### Quality Settings
1. Navigate to **Settings -> Video** (or via in-player Gear menu during playback).
2. **Local Quality:** `Original` / `Maximum`.
3. Ensure **Direct Play** and **Direct Stream** are both ticked.

### Cache Flush Script (PowerShell on Windows)
Run if the client retains stale routing information or TLS state:
```powershell
Stop-Process -Name "Plex HTPC" -ErrorAction SilentlyContinue
Remove-Item -Recurse -Force "$env:LOCALAPPDATA\Plex HTPC\Cache" -ErrorAction SilentlyContinue
Remove-Item -Recurse -Force "$env:LOCALAPPDATA\Plex HTPC\web-client-storage" -ErrorAction SilentlyContinue
Start-Process "$env:PROGRAMFILES\Plex\Plex HTPC\Plex HTPC.exe"
```

---

## 5. Verification & Diagnostics

### Monitoring Transcoder Status (LXC)
```bash
tail -fn 50 "/var/lib/plexmediaserver/Library/Application Support/Plex Media Server/Logs/Plex Media Server.log" | grep -iE 'transcode|mde|decision'
```

### Plex Web Dashboard Verification
Access `http://192.168.3.57:32400/web` and open the **Dashboard**:
* Direct Play: Shows `Direct Play` for both Video and Audio streams with 0% transcode overhead.
* Hardware Transcode: Shows `Video 1080p (H.264) (hw) -> SD (H264) — Transcode (hw)`. Both sides must show `(hw)` to verify hardware acceleration on the Intel GPU.
