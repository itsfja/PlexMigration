Plex Migration Guide: Windows VM (`WinPlex`) to Proxmox LXC on `pve1`
This guide details the procedure for decommissioning the virtualised Windows Plex instance (`WinPlex` on VM 101, `pve2`) and deploying a native, unprivileged Debian LXC container on `pve1` with Intel QuickSync GPU pass-through, host-level NFS/SMB bind mounts, and RAM-backed transcoding.
---
Architecture Overview
Source: VM 101 (`WinPlex` running on `pve2` alongside NZBGet / the *arr stack)
Destination Host: `pve1`
Guest Type: Unprivileged Debian 12 LXC
Storage Mount Strategy: Hypervisor-level mount (`/mnt/media`) passed directly to container via `mp0` bind mount
Transcode Cache: Shared memory buffer (`/dev/shm` in RAM) to eliminate disk I/O bottlenecks
---
Phase 1: Preparation on Source (`WinPlex` VM 101)
Open Plex Web on `WinPlex`.
Navigate to Settings -> Library and disable:
Scan my library automatically
Run a partial scan when changes are detected
Empty trash automatically after every scan (critical: prevents library loss if media mounts disconnect during migration)
Stop the Plex Media Server service in Windows:
Open PowerShell as Administrator:
```powershell
     Stop-Process -Name "Plex Media Server" -Force
     ```
Back up user metadata and database files:
Plex app data resides at:  
`%LOCALAPPDATA%\Plex Media Server\`
Create an archive or copy the directory (specifically `Plug-in Support/Databases/com.plexapp.plugins.library.db` and the `Media` / `Metadata` folders) to your backup storage or NAS share.
---
Phase 2: Host Storage Configuration on `pve1`
Mount your NAS media shares at the hypervisor level on `pve1` to provide direct filesystem access without virtual network interface overhead.
SSH into `pve1` as `root`.
Create mount points:
```bash
   mkdir -p /mnt/nas/media
   ```
Mount the media export via `/etc/fstab` (using NFSv4 recommended for performance):
```text
   # /etc/fstab entry on pve1
   <NAS_IP>:/path/to/media  /mnt/nas/media  nfs  defaults,ro,nolock,noatime,rsize=1048576,wsize=1048576  0  0
   ```
(Note: Use `ro` or `rw` depending on whether Plex needs write permissions for subtitle downloads or local file deletions.)
Mount all filesystems:
```bash
   mount -a
   ls -la /mnt/nas/media
   ```
---
Phase 3: Create and Configure the Plex LXC on `pve1`
1. Create the Container
In the Proxmox VE Web UI on `pve1` (or via CLI):
CT ID: e.g., `201`
Hostname: `plex-lxc`
Template: Debian 12 Standard (or Ubuntu 24.04 LTS)
Unprivileged container: Yes
Cores: 4
Memory: 2048 MB to 4096 MB (Swap: 1024 MB)
Root Disk: 20 GB – 30 GB on local NVMe/SSD storage
Network: Attach to your primary LAN/Management bridge (assign static IP or DHCP reservation)
2. Configure Intel QuickSync iGPU Pass-Through
Identify the major and minor device nodes for render and video devices on `pve1`:
```bash
ls -la /dev/dri
```
Standard output typically shows `card0` (major 226, minor 0) and `renderD128` (major 226, minor 128).
Edit the container configuration file on `pve1` at `/etc/pve/lxc/<CT_ID>.conf`:
```bash
nano /etc/pve/lxc/<CT_ID>.conf
```
Append the following configuration at the end:
```ini
# Intel QuickSync GPU Passthrough
lxc.cgroup2.devices.allow: c 226:* rwm
lxc.mount.entry: /dev/dri dev/dri none bind,optional,create=dir
lxc.mount.entry: /dev/dri/renderD128 dev/dri/renderD128 none bind,optional,create=file

# Hypervisor Media Bind Mount
mp0: /mnt/nas/media,mp=/media,ro=1

# Container permissions for render group (gid 107/108 match host 'render' group)
lxc.idmap: u 0 100000 65536
lxc.idmap: g 0 100000 44
lxc.idmap: g 44 44 1
lxc.idmap: g 45 100045 62
lxc.idmap: g 107 107 1
lxc.idmap: g 108 100108 65428
```
(Verify host GIDs for `video` and `render` groups via `getent group video render` on `pve1` and align the configuration accordingly).
---
Phase 4: Install Plex in the LXC
Start the container:
```bash
   pct start <CT_ID>
   pct enter <CT_ID>
   ```
Update packages and install dependencies:
```bash
   apt update && apt install -y curl gnupg2 sudo vainfo
   ```
Verify Intel hardware acceleration inside the container:
```bash
   vainfo
   ```
(Ensure the Intel VA-API driver loads without initialization errors).
Add the official Plex repository and install Plex Media Server:
```bash
   curl https://downloads.plex.tv/plex-keys/PlexSign.key | gpg --dearmor -o /etc/apt/trusted.gpg.d/plex.gpg
   echo "deb [signed-by=/etc/apt/trusted.gpg.d/plex.gpg] https://downloads.plex.tv/repo/deb public main" > /etc/apt/sources.list.d/plexmediaserver.list
   apt update && apt install -y plexmediaserver
   ```
---
Phase 5: Transcoding to RAM and Network Optimization
Configure Transcoder Directory:
Open Plex Web at `http://<NEW_LXC_IP>:32400/web`.
Go to Settings -> Transcoder.
Set Transcoder temporary directory to:
```text
     /dev/shm
     ```
Set Transcoder default throttle buffer to `120`.
Check Use hardware acceleration when available.
Check Use hardware-accelerated video encoding.
Define Subnets for Direct Play:
Go to Settings -> Network.
Set LAN Networks to include all local and IoT subnets:
```text
     192.168.1.0/24,192.168.3.0/24,192.168.10.0/24
     ```
Set Enable Relay to Disabled to prevent 1–2 Mbps proxy fallbacks.
Map Media Folders:
In Plex, point your libraries to `/media/...` reflecting your bound NAS paths.
Verify direct play performance across clients.
---
Phase 6: Decommission `WinPlex` on `pve2`
Once the LXC instance on `pve1` is verified and playing correctly, power down VM 101 on `pve2`:
```bash
   qm stop 101
   ```
Uncheck Start at boot on VM 101 in Proxmox.
Update Port Forwarding or Cloudflare Tunnel routes (if external access is enabled) to point to the new IP address of the LXC container on `pve1`.
Delete or archive VM 101 once testing is complete to reclaim system memory and storage on `pve2`.
