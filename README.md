md_content = """# Plex Media Server (LXC) & Plex HTPC Cross-VLAN Deployment Runbook

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
customConnections="[http://192.168.3.57:32400](http://192.168.3.57:32400)"
SecureConnections="0"
lanNetworks="192.168.3.0/24,192.168.10.0/24"
allowedNetworks="192.168.3.0/24,192.168.10.0/24"
TranscoderTempDirectory="/dev/shm"
HardwareDevicePath="8086:9bca:8086:2081@0000:00:02.0"
