# Proxmox Secure Gateway Lab

## 📌 Project Overview
This repository documents the architecture of a secure, home-hosted virtualization environment running on **Proxmox VE**. The project focuses on implementing "Zero Trust" networking principles, enabling hardware acceleration within containers, and optimizing storage reliability for hybrid (NVMe + USB DAS) setups.

**Hardware:**
* **Host:** Dell OptiPlex Micro (Intel Core i5-8500T)
* **Tier 1 Storage:** NVMe SSD (OS, Cache, AppData)
* **Tier 2 Storage:** 4TB External DAS via USB 3.0 (Mass Storage)
* **GPU:** Intel UHD Graphics 630 (Passed through for hardware offloading)

## 🏗️ Key Engineering Implementations

### 1. Network Isolation (Sidecar Pattern)
To secure external traffic, the system creates a "Black Hole" network architecture using `gluetun`.
* **Implementation:** Sensitive containers share the network stack of the VPN container (`network_mode: service:gluetun`).
* **Protocol:** WireGuard with **NAT-PMP** enabled to handle dynamic port forwarding requirements from the VPN provider (ProtonVPN).
* **Security:** Acts as a strictly enforced Kill Switch; if the tunnel drops, all WAN connectivity ceases immediately.

### 2. LXC Resource Passthrough
To enable high-performance media processing and VPN tunneling capabilities within unprivileged LXC containers, I manually modified the container configuration hooks.

*Configuration snippet (`/etc/pve/lxc/100.conf`):*

    # Intel QuickSync Passthrough (Hardware Transcoding)
    lxc.cgroup2.devices.allow: c 226:0 rwm
    lxc.cgroup2.devices.allow: c 226:128 rwm
    lxc.mount.entry: /dev/dri dev/dri none bind,optional,create=dir

    # TUN/TAP Device Passthrough (Required for VPN/Tailscale inside LXC)
    lxc.cgroup2.devices.allow: c 10:200 rwm
    lxc.mount.entry: /dev/net/tun dev/net/tun none bind,create=file

### 3. Storage Latency Mitigation
Using an external USB enclosure (DAS) introduced aggressive hardware sleep states, causing I/O timeouts during idle periods. I developed a heartbeat daemon to mitigate this controller behavior.

*Custom Cron Job:*

    # Writes a zero-byte file every 5 minutes to prevent controller spin-down
    */5 * * * * touch /mnt/hdd/.keepalive

### 4. Unified Namespace Strategy
To optimize I/O, the file system maps different physical disks (NVMe & HDD) into a single logical volume structure (`/data`). This allows applications to perform **Atomic Moves (Hardlinks)**, preventing unnecessary read/write operations on the USB bus.

## 🛠️ Tech Stack
* **Hypervisor:** Proxmox VE (LXC & KVM)
* **Container Engine:** Docker & Portainer
* **Networking:** WireGuard, Tailscale (Remote Access)

---
*Infrastructure maintained by [Jesper Nilsson](https://github.com/JeNilSE)*
