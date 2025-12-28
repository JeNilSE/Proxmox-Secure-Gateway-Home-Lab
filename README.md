# Proxmox Secure Gateway Lab

## 🌄 Project Overview
This repository documents a secure, home-hosted virtualization environment running on **Proxmox VE**. The project focuses on implementing Zero Trust networking principles, enabling hardware acceleration within containers, and optimizing storage reliability for hybrid (NVMe + USB DAS) setups.

**Hardware:**
* **Host:** Dell OptiPlex Micro (Intel Core i5-8500T)
* **Main storage:** 1TB NVMe SSD (OS, Cache, AppData)
* **DAS:** 4TB External DAS via USB 3.0 (Mass Storage)
* **GPU:** Intel UHD Graphics 630 (Passed through for hardware offloading)

## 🔑 Key Implementations

### 1. Network Isolation
To secure external traffic, the system creates a "Black Hole" network architecture using `gluetun`.
* **Implementation:** Sensitive containers share the network stack of the VPN container (`network_mode: service:gluetun`).
* **Protocol:** WireGuard with **NAT-PMP** enabled to handle dynamic port forwarding requirements from the VPN provider (ProtonVPN).
* **Security:** Acts as a strictly enforced Kill Switch. If the tunnel drops, all WAN connectivity stops.

### 2. Intel QuickSync Passthrough
To enable decently high media processing and VPN tunneling capabilities within LXC containers, I manually modified the container configuration.

*Configuration snippet (`/etc/pve/lxc/100.conf`):*

    # Intel QuickSync Passthrough (Hardware Transcoding)
    lxc.cgroup2.devices.allow: c 226:0 rwm
    lxc.cgroup2.devices.allow: c 226:128 rwm
    lxc.mount.entry: /dev/dri dev/dri none bind,optional,create=dir

    # TUN/TAP Device Passthrough (Required for VPN/Tailscale inside LXC)
    lxc.cgroup2.devices.allow: c 10:200 rwm
    lxc.mount.entry: /dev/net/tun dev/net/tun none bind,create=file

### 3. Keepalive Cron Job
Using an external USB enclosure (DAS) required me to setup a heartbeat daemon that touches an empty file on the HDD so the drive does not go to sleep and is always on.

*Custom Cron Job:*

    # Writes a zero-byte file every 5 minutes to prevent controller spin-down
    */5 * * * * touch /mnt/hdd/.keepalive

### 4. Atomic Moves
The file system maps different physical disks (NVMe & HDD) into a single logical volume structure (`/data`). This allows applications to perform **Atomic Moves (Hardlinks)**, preventing unnecessary read/write operations on the drive connected with USB.

## 📚 Tech Stack
* **Hypervisor:** Proxmox VE (LXC & KVM)
* **Container Engine:** Docker & Portainer
* **Networking:** WireGuard, Tailscale (Remote Access)

---
*Infrastructure maintained by [Me](https://github.com/JeNilSE)*
