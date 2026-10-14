<div align="center">

# 🏠 Home Server

**A self-hosted Unraid home server: a media-automation stack, Cloudflare Tunnel plus Tailscale networking, Datadog monitoring, and a Dokploy VM for side projects.**

`Unraid` · `Docker` · `*arr stack` · `Jellyfin` · `Dokploy` · `Datadog` · `Jonsbo N4 NAS`

</div>

---

## 📖 Table of Contents

- [Overview](#-overview)
- [Architecture](#-architecture)
- [Hardware](#-hardware)
  - [System summary](#system-summary)
  - [Cooling & airflow](#-cooling--airflow)
  - [Storage layout](#storage-layout)
- [Software stack](#-software-stack)
  - [Host & virtualization](#host--virtualization)
  - [Containers](#containers)
  - [Dokploy](#dokploy)
- [The *arr* media stack](#-the-arr-media-stack)
  - [How the automation flows](#how-the-automation-flows)
- [Networking & remote access](#-networking--remote-access)
- [Storage & media library layout](#-storage--media-library-layout)
- [Backups & data safety](#-backups--data-safety)
- [Maintenance & operations](#-maintenance--operations)
- [Reproducing this build](#-reproducing-this-build)

---

## 🔭 Overview

The host runs **[Unraid OS](https://unraid.net)** (7.3), a NAS/server OS built around a flexible, parity-protected storage array with first-class Docker and VM support. On top of it, the server does two jobs:

1. **Media automation and services.** The *"arr"* stack runs directly in **Unraid's Docker**, automatically finding, downloading, organizing, and streaming media through **Jellyfin**. A **`cloudflared`** container publishes selected services via Cloudflare Tunnel, **Tailscale** provides private remote admin, and a **Datadog agent** ships metrics and logs for monitoring.
2. **App hosting.** A single **Ubuntu VM** runs [**Dokploy**](#dokploy), a self-hostable Heroku/Vercel, which deploys various personal side projects.

Storage is a **3× 12 TB** parity-protected Unraid array fronted by a 1 TB NVMe **cache** drive. Writes land on the SSD and Unraid's **Mover** flushes them to the array nightly. Everything lives in a compact [Jonsbo N4](#system-summary) NAS chassis.

---

## 🗺 Architecture

```mermaid
flowchart TB
  subgraph HOST["🖥 Unraid host · Jonsbo N4"]
    HW["Core Ultra 7 265K · 32 GB DDR5"] --> UNRAID["Unraid OS 7.3"]

    subgraph DOCKER["🐳 Unraid Docker"]
      ARR["*arr stack<br/>Prowlarr · Sonarr · Radarr · Bazarr<br/>FlareSolverr · Deluge (VPN) · Cleanuparr"]
      JF["Jellyfin"]
      CF["cloudflared<br/>(Cloudflare Tunnel)"]
      DD["Datadog agent"]
    end

    subgraph KVM["🧩 Ubuntu VM (KVM)"]
      DOK["Dokploy<br/>→ side projects"]
    end

    UNRAID --> DOCKER & KVM
  end

  subgraph STORE["💾 Storage"]
    CACHE["Cache · 1 TB NVMe (btrfs)"]
    ARRAY["Array · 2× 12 TB data + 1 parity (xfs)"]
    CACHE -->|"Mover · nightly"| ARRAY
  end

  ARR --> CACHE
  JF --> ARRAY
  CF --> WAN((Internet))
  DOK --> WAN
```

---

## 🔩 Hardware

All component specs below are verified against manufacturer/retailer documentation and the live system.

### System summary

| Component | Spec | Notes |
|---|---|---|
| **CPU** | Intel Core Ultra 7 265K | 20 cores (8 P + 12 E), 20 threads (no Hyperthreading).<br/>3.9 GHz base, 5.5 GHz boost. LGA1851.<br/>125 W base, 250 W max turbo.<br/>Integrated Intel Xe graphics with Quick Sync, used for **Jellyfin hardware transcoding** (HEVC, H.264, AV1). |
| **CPU cooler** | Noctua NH-L9x65 (Brown, 65 mm) | Premium low-profile air cooler. LGA1851 supported.<br/>CPU power-limited in BIOS to stay within its envelope (see [cooling & airflow](#-cooling--airflow)). |
| **Motherboard** | MSI PRO B860M-A WIFI (MS-7E42) | Intel B860 chipset, Micro-ATX, LGA1851.<br/>4× DDR5 DIMM (up to 256 GB). Onboard 2.5 GbE and Wi-Fi. |
| **Memory** | G.Skill Ripjaws S5 32 GB (2×16 GB) | DDR5-5600, CL36. |
| **Cache drive** | Samsung 990 EVO Plus 1 TB | M.2-2280 NVMe. Hybrid PCIe 4.0 ×4 / 5.0 ×2 (~7,150 MB/s ceiling).<br/>Unraid cache (btrfs): appdata plus write landing zone. |
| **Storage array** | 3× 12 TB 3.5" HDD | Mixed NAS drives (Seagate IronWolf, IronWolf Pro, HGST Ultrastar).<br/>Unraid array: 1 parity plus 2 data (~24 TB usable, single-drive fault tolerance). |
| **Boot device** | USB flash (~119 GB) | Unraid boots from and is licensed to the USB stick. |
| **Power supply** | Lian Li SP850 | 850 W, 80+ Gold, fully modular, **SFX** form factor. |
| **Case** | Jonsbo N4 | Micro-ATX NAS chassis. 6× 3.5" (4 hot-swap, 2 fixed) plus 2× 2.5" = 8 bays.<br/>Takes SFX PSU. ⚠️ Poor stock ventilation (see [airflow mod](#-cooling--airflow)). |
| **Case fans** | Noctua NF-A12x25 PWM chromax.black.swap (120 mm)<br/>2× ARCTIC P12 Slim PWM PST (120 mm) | Slim front fans added via a 3D-printed adapter (see below). |

### ❄️ Cooling & airflow

Two cooling considerations, both addressed:

**1. CPU thermal envelope.** The Noctua NH-L9x65 is a low-profile cooler designed for roughly 35 to 95 W SFF builds, while the Core Ultra 7 265K can pull up to **250 W** at full turbo. To keep temperatures in check on an always-on server, the CPU is **power-limited (PL1/PL2) in BIOS** rather than left at stock turbo. This trades away peak burst performance the server does not need for a quiet, cool box. The cooler fits the LGA1851 socket fine; this is purely a heat-headroom choice.

**2. Case ventilation.** The Jonsbo N4 ships with minimal airflow: a single fan in the **bottom** of the chassis and no intake up front. To pull cool air across the drives and components, **two slim 120 mm fans (ARCTIC P12 Slim PWM PST)** were added to the **front**, mounted with a **3D-printed slim-fan front adapter** ([SLIM Fans Front Adapter for Jonsbo N4 on eBay](https://www.ebay.com/itm/406224064291)). The P12 Slim's extra-thin profile is what makes it fit the front of this case where a standard 25 mm fan would not.

<div align="center">
  <img src="assets/jonsbo-n4-build.png" alt="The home server: a wood-fronted compute case stacked on top of the black Jonsbo N4 NAS chassis with its perforated front panel" width="600" />
  <br />
  <sub>The Jonsbo N4 (black, bottom) with its perforated front panel.</sub>
</div>

### Storage layout

Unraid pools the drives into a single parity-protected array with an SSD cache in front:

| Device | Drive | Size | Role | FS | Mounted as |
|---|---|---|---|---|---|
| Cache | Samsung 990 EVO Plus | 1 TB NVMe | appdata, Docker, VM image, write cache | btrfs | `/mnt/cache` |
| Parity | Seagate IronWolf | 12 TB | Parity (single-drive fault tolerance) | n/a | n/a |
| Disk 1 | HGST Ultrastar | 12 TB | Array data | xfs | `/mnt/disk1` |
| Disk 2 | Seagate IronWolf Pro | 12 TB | Array data | xfs | `/mnt/disk2` |
| Boot | USB flash | ~119 GB | Unraid OS (boot, license) | vfat | n/a |
| User share | n/a | ~24 TB | Unified view spanning array plus cache | n/a | `/mnt/user` |

- **Single parity.** The array tolerates **one** drive failure (a second parity slot exists but is unassigned).
- **Cache plus Mover.** Writes hit the NVMe cache first; the **Mover** flushes them to the array on a nightly schedule.

---

## 💿 Software stack

### Host & virtualization

| Layer | Detail |
|---|---|
| Host OS | **Unraid OS 7.3** (boots from USB) |
| Containers | Unraid's built-in **Docker**, running the media stack, `cloudflared`, and the Datadog agent |
| Virtual machine | An **Ubuntu** VM (KVM) dedicated to Dokploy |

> **Why this split?** Unraid handles storage, Docker, and VMs natively, so the media stack runs as lightweight containers right on the host. Dokploy lives in its own Ubuntu VM so deploying and redeploying side projects cannot destabilize the always-on media and storage services.

### Containers

Everything on Unraid Docker falls into three groups:

| Group | Containers | Role |
|---|---|---|
| **Media stack** | Prowlarr<br/>Sonarr<br/>Radarr<br/>Bazarr<br/>FlareSolverr<br/>Deluge (VPN)<br/>Cleanuparr<br/>Jellyfin<br/>Jellyseerr | Media automation and streaming. Detailed in [the *arr* media stack](#-the-arr-media-stack). |
| **Networking** | `cloudflared` | Cloudflare Tunnel publishes selected services with no open inbound ports. Tailscale handles admin (see [Networking](#-networking--remote-access)). |
| **Monitoring** | Datadog agent | Host and container metrics and logs shipped to Datadog. |

The Ubuntu VM runs **Dokploy**, which in turn hosts various personal side projects (see [Dokploy](#dokploy)).

### Dokploy

[**Dokploy**](https://dokploy.com) is an open-source, self-hostable PaaS. It runs on top of Docker and gives a web dashboard to deploy applications and databases, manage environment variables, wire up domains and TLS, and watch logs, without hand-writing Compose files for everything. It is the control plane for the app-hosting half of this server.

| | |
|---|---|
| Runs on | Ubuntu VM (KVM) on the Unraid host |
| Role | Deploys and manages various personal side projects (websites, an Obsidian sync server, and other small apps) |

---

## 🎬 The *arr* media stack

A family of self-hosted tools that each do one job and hand off to the next, automating the whole pipeline from request to stream. The setup follows the **[TRaSH Guides](https://trash-guides.info/)** conventions for naming, quality profiles, and folder structure.

| App | Container / image | What it does (in plain language) |
|---|---|---|
| **Prowlarr** | `binhex/arch-prowlarr` | Indexer manager. The single place to configure trackers and indexers; syncs them out to Sonarr and Radarr so you do not set them up app-by-app. |
| **Sonarr** | `binhex/arch-sonarr` | TV automation. Monitors shows, grabs new episodes as they release, then renames and files them into the library. |
| **Radarr** | `binhex/arch-radarr` | Same idea as Sonarr, but for movies. |
| **Bazarr** | `linuxserver/bazarr` | Finds and downloads subtitles for whatever Sonarr and Radarr pull in. |
| **FlareSolverr** | `binhex/arch-flaresolverr` | Proxy that solves Cloudflare and CAPTCHA challenges so indexers stay reachable. |
| **Deluge (plus VPN)** | `binhex/arch-delugevpn` | The BitTorrent download client. Its traffic routes through a **built-in PIA VPN**, so everything is tunneled and downloads stop rather than leak if the VPN drops. |
| **Cleanuparr** | `cleanuparr/cleanuparr` | Housekeeping for the queue. Auto-removes stalled, failed, or orphaned downloads and blocklists bad releases so the *arr* apps keep retrying cleanly. |
| **Jellyfin** | `jellyfin/jellyfin` | Open-source media server. Catalogs the library and streams it to your devices, with Quick Sync hardware transcoding on the CPU's iGPU. |
| **Jellyseerr** | `fallenbagel/jellyseerr` | Request front-end. A friendly UI where users request titles, which get handed to Sonarr and Radarr. |

> Most *arr* apps use [**binhex**](https://github.com/binhex) image variants; Bazarr uses [**LinuxServer**](https://www.linuxserver.io/).

### How the automation flows

```mermaid
sequenceDiagram
  participant U as User / Jellyseerr
  participant ARR as Sonarr / Radarr
  participant P as Prowlarr
  participant DL as Deluge (via PIA VPN)
  participant M as Jellyfin

  U->>ARR: Request a title
  ARR->>P: Search configured indexers
  P-->>ARR: Best matching release
  ARR->>DL: Send to download client
  DL-->>ARR: Download complete
  ARR->>ARR: Rename and import into library
  ARR->>M: Library updated
  M-->>U: Available to stream
```

---

## 🌐 Networking & remote access

| Item | Detail |
|---|---|
| Router / Wi-Fi | **ASUS ZenWiFi AX (XT8, RT-AX95Q)** — AX6600 tri-band Wi-Fi 6 mesh, two nodes, AiMesh. Main node: 1× 2.5 GbE WAN, 3× 1 GbE LAN |
| Internet | 1 Gbps fiber, ONT → XT8 2.5G WAN |
| LAN | 2.5 GbE onboard, bonded. Links at 1 GbE, since the XT8 LAN ports are gigabit |
| Published services | **Cloudflare Tunnel** via a `cloudflared` container. Selected services are reachable externally with **no inbound ports opened** on the router. |
| Remote admin | **Tailscale**, a private mesh VPN for administering the box remotely |
| TLS / certificates | Terminated at Cloudflare's edge |

---

## 🗂 Storage & media library layout

Unraid exposes everything through **user shares** (`/mnt/user/...`) that span the cache and the array. A representative layout:

```text
/mnt/user
├── media/        # movies, tv (served by Jellyfin from the array)
├── downloads/    # active downloads (kept on cache for speed)
├── appdata/      # per-container config (kept on cache)
└── system/       # docker.img / libvirt.img
```

- **Cache plus Mover.** Writes land on the 1 TB NVMe cache for speed; Unraid's **Mover** flushes media to the parity-protected array on a nightly schedule. `appdata` and active downloads stay cache-resident.
- **Hardlinks and atomic moves.** The download and media paths live inside the **same Unraid user share** so imports are instant hardlinks instead of slow cross-share copies.

---

## 💾 Backups & data safety

Unraid **parity** protects the array against a single-drive failure, but parity is not a backup. The data that actually hurts to lose is configuration, not media:

- **`appdata` (container configs).** The per-container settings for the whole media stack, backed up so the stack can be rebuilt without reconfiguring each app from scratch.
- **Dokploy state.** The VM and its deployment configuration.
- **Media files** are intentionally **not** backed up. They are re-acquirable, and backing up tens of terabytes is not practical.

---

## 🛠 Maintenance & operations

Most day-to-day management happens in the **Unraid web UI** (Dashboard, Main, Docker, and VMs tabs). The commands below harvest the same facts over SSH for documentation.

```bash
# Hardware (run on the Unraid host)
lscpu                                              # CPU
free -h                                            # RAM
lsblk -o NAME,SIZE,TYPE,MODEL,MOUNTPOINT           # disks
cat /etc/unraid-version && uname -r                # Unraid plus kernel

# Containers (Unraid host)
docker ps --format 'table {{.Names}}\t{{.Image}}\t{{.Status}}'
```

**Routine tasks:**

- **Update containers.** Unraid **Docker** tab to check for updates and apply (host services); **Dokploy** redeploy for VM-hosted side projects.
- **Array and Mover.** The Mover runs nightly on a schedule (Settings, then Scheduler); parity checks run on Unraid's schedule.
- **Monitoring.** **Datadog** (agent container) for metrics and logs, plus the built-in Unraid Dashboard.

---

## 🧱 Reproducing this build

High-level steps to rebuild from scratch:

1. Flash **Unraid** to a USB stick, boot, apply the license, and configure the array: assign **1 parity plus 2 data** 12 TB drives and the 1 TB NVMe as **cache** (btrfs).
2. Enable Docker; install the media stack (**Prowlarr, Sonarr, Radarr, Bazarr, FlareSolverr, Deluge+VPN, Cleanuparr, Jellyfin, Jellyseerr**), `cloudflared`, and the **Datadog agent** via Unraid Community Apps. Follow [TRaSH Guides](https://trash-guides.info/) for quality profiles and folder structure.
3. Set up the **Cloudflare Tunnel** (publish services) and **Tailscale** (remote admin).
4. Create the **Ubuntu VM** (KVM); install Docker plus **Dokploy**.
5. Redeploy side projects through the Dokploy dashboard.
6. Restore `appdata` and Dokploy state from backup.

---

<div align="center">
<sub>Self-hosted with ☕ and a Jonsbo N4.</sub>
</div>
