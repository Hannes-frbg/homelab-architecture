# 🏰 Proxmox VE Homelab Architecture & Cluster Guide

[![Proxmox VE](https://img.shields.io/badge/Proxmox_VE-8.4-E57000?logo=proxmox&logoColor=white)](https://www.proxmox.com)
[![Proxmox Backup Server](https://img.shields.io/badge/PBS-4.2-E57000?logo=proxmox&logoColor=white)](https://www.proxmox.com/en/proxmox-backup-server)
[![Debian](https://img.shields.io/badge/Debian-12_Bookworm-A81D33?logo=debian&logoColor=white)](https://www.debian.org)
[![Git Automation](https://img.shields.io/badge/DevOps-Git_AutoSync-181717?logo=git&logoColor=white)](#-automatisierung--gitops-sync)
[![Security Status](https://img.shields.io/badge/Security-Sanitized_%26_2FA-2ea44f)](#-sicherheitskonzept--härtung)

Willkommen in der Architektur-Dokumentation des **Proxmox Homelab Clusters**.  
Dieses Repository bietet eine schnelle, visuelle Übersicht über die beiden physischen Nodes, virtualisierte Gaming- und Service-Stacks, das 2FA-Sicherheits-Gateway sowie die automatisierte Backup- und Git-Infrastruktur.

---

## ⚡ Schnellübersicht (TL;DR)

- **2 Physische Hypervisor-Nodes** im Heimnetz (`192.168.178.0/24`)
- **Core Compute Node (PVE 1):** AMD Ryzen 5 5600GT (6 Kerne / 12 Threads) mit 32 GB RAM, 1 TB NVMe/SATA SSD + 1 TB HDD
- **Backup & Gateway Node (PVE 2):** Intel Core i5-7500T (4 Kerne / 4 Threads, Low-Power USFF) mit 8 GB RAM, 512 GB SSD
- **Modernisierter Gaming-Stack:** Pelican Panel & Wings auf **VM 110** (Velocity Modern Forwarding mit Port 25577 Proxy)
- **Zentraler Backup-Server:** Nativer **Proxmox Backup Server (PBS 4.2)** auf Node 2 (`:8007`) mit datei- & blockbasierter Deduplizierung
- **2FA-Zugangssicherung:** Node.js Web-Portal auf **CT 100** via Cloudflare Tunnel mit Google Authenticator (TOTP)
- **Vollautomatisches GitOps-Backup:** Täglicher systemd-Timer auf PVE 1 mit Secret-Scrubbing & Git-Push

---

## 🗺️ Visuelle Cluster-Architektur

```mermaid
flowchart TD
    subgraph WAN ["🌐 Externer Zugang (Internet)"]
        CF["Cloudflare Zero Trust Tunnel"]
        SRV["Minecraft Domain (Port 25577)"]
    end

    subgraph LAN ["🏠 Lokales Netzwerk (192.168.178.0/24)"]
        FRITZ["FritzBox Gateway & DNS<br/>192.168.178.1"]
    end

    subgraph PVE1 ["🖥️ Node 1: PVE 1 (192.168.178.169)<br/>AMD Ryzen 5 5600GT (6C/12T) | 32 GB RAM<br/>256 GB SSD (OS) | 1 TB SSD (ZFS) | 1 TB HDD"]
        direction TB
        VM110["<b>VM 110: Pelican Gaming Stack</b><br/>IP: 192.168.178.94<br/>• Velocity Proxy (25577)<br/>• Purpur SMP (25565)<br/>• Test Server (25566)"]
        VM104["<i>VM 104: Legacy Minecraft (61 GB)</i><br/>(Altsystem - Abschaltung empfohlen)"]
        SYNC["⚙️ <b>Git-Sync Daemon & Secret Sanitizer</b><br/>systemd-Timer: 05:00 Uhr täglich"]
    end

    subgraph PVE2 ["🖥️ Node 2: PVE 2 (192.168.178.2)<br/>Intel Core i5-7500T (4C/4T @ 2.7GHz) | 8 GB RAM<br/>512 GB SSD (LVM-Thin / PBS Storage)"]
        direction TB
        PBS["🛡️ <b>Proxmox Backup Server 4.2</b><br/>Nativ auf Port 8007 | /mnt/pbs-storage (280 GB)"]
        CT100["<b>CT 100: Auth-Portal (2FA)</b><br/>IP: 192.168.178.71<br/>Node.js + Google Authenticator"]
        CT101["<i>CT 101: Pterodactyl Legacy</i><br/>IP: 192.168.178.115"]
    end

    CF -->|HTTPS| CT100
    SRV -->|TCP 25577| VM110
    PVE1 -->|SSH Cluster Trust & Storage API| PVE2
    PVE1 -->|Nächtliches Backup| PBS
    PVE2 -->|Lokale Sicherung| PBS
    SYNC -.->|qm guest exec| VM110
    SYNC -.->|Auto-Push| GH["GitHub: homelab-configs & homelab-architecture"]
```

---

## 🖥️ Physische Nodes & Echte Hardware-Spezifikationen

| Komponente | Node 1 (`PVE 1` — Core Compute) | Node 2 (`PVE 2` — Backup & Gateway) |
| :--- | :--- | :--- |
| **Hostname / IP** | `pve1` / `192.168.178.169` | `pve2` / `192.168.178.2` |
| **Prozessor (CPU)** | **AMD Ryzen 5 5600GT with Radeon Graphics**<br/>• 6 physische Kerne / 12 Threads<br/>• Zen 3 Architektur (7nm, Cezanne)<br/>• 3.60 GHz Basistakt / bis zu 4.65 GHz Boost | **Intel Core i5-7500T CPU @ 2.70GHz**<br/>• 4 physische Kerne / 4 Threads<br/>• Kaby Lake Architektur (14nm)<br/>• 35W Low-Power TDP (Ultra-Slim / Tiny Form Factor) |
| **Arbeitsspeicher (RAM)** | **32 GB DDR4** (30 GiB nutzbar) | **8 GB DDR4** (7.6 GiB nutzbar) |
| **Speicher-Konfiguration** | • **SSD 1 (`sda`):** 256 GB SSD (OS root `local` + `local-lvm`)<br/>• **SSD 2 (`sdb`):** 1 TB SSD (ZFS Pool `ssd-1tb` für VMs)<br/>• **HDD 1 (`sdc`):** 1 TB Western Digital (ZFS Pool `Hdd` + `hdd-backup`) | • **SSD 1 (`sda`):** 512 GB SSD (447 GiB nutzbar)<br/>  - OS root: 96 GB (`local`)<br/>  - LVM-Thin Pool: 320 GB (`local-lvm`)<br/>  - PBS Datastore: 280 GB Mount (`/mnt/pbs-storage`) |
| **Storage-Architektur** | ZFS Pools (`ssd-1tb`, `Hdd`) + LVM-Thin | Schlankes LVM / LVM-Thin (kein ZFS-Overhead, optimal für 8 GB RAM) |
| **Primäre Rolle** | Hochperformante Game-Server (Pelican), Core VMs | Deduplizierendes Backup-Ziel (PBS 4.2) & 2FA Auth-Portal |

---

## 📦 Gäste, Virtualisierung & Dienste

| VM/CT ID | Typ | Name / Hostname | IP-Adresse | Node | vCPU / RAM | Zweck & Zustand |
| :---: | :---: | :--- | :--- | :---: | :---: | :--- |
| **110** | QEMU VM | `pelican` | `192.168.178.94` | PVE 1 | 10 vCPUs / 16 GB | **Produktiv:** Pelican Panel, Wings & Minecraft-Server (Purpur & Velocity) |
| **104** | QEMU VM | `Minecraftserver` | DHCP | PVE 1 | 6 vCPUs / 12 GB | *Legacy:* Altes Serversystem (61 GB Speicher), Stilllegung empfohlen |
| **100** | LXC | `auth-portal` | `192.168.178.71` | PVE 2 | 2 Cores / 1 GB | **Security:** 2FA-Webportal (Node.js) mit Cloudflare Tunnel |
| **101** | LXC | `pterodactyl` | `192.168.178.115` | PVE 2 | 2 Cores / 1 GB | *Migration:* Altes Pterodactyl-Panel, Daten migriert auf VM 110 |
| **Nativ** | Dienst | `Proxmox Backup Server` | `192.168.178.2:8007` | PVE 2 | Host-Dienst | **Datensicherung:** Deduplizierender PBS-Datenspeicher für alle Gäste |

---

## 🎮 Minecraft Gaming-Architektur (VM 110)

Der Minecraft-Betrieb läuft vollständig containerisiert und isoliert unter **Pelican Panel & Wings**:

```
Spieler-Client
      │
      ▼
Velocity Proxy (Port 25577) ──[Modern BungeeGuard Forwarding]──┐
      │                                                        │
      ├───────────────────────────────┐                        │
      ▼                               ▼                        ▼
Purpur SMP (Port 25565)      Test-Server (Port 25566)    Event-Server (optional)
```

- **Sicherheit:** Velocity kommuniziert mit den Unter-Servern via `modern` Player-Info-Forwarding und einem 32-stelligen kryptografischen Secret. Direkte Spieler-Verbindungen auf die Backend-Ports werden abgewiesen.
- **Speicherorte:** Die Server-Volumes liegen unter `/var/lib/pelican/volumes/<UUID>/`.

---

## 🛡️ Sicherheitskonzept & Härtung

1. **Google Authenticator 2FA Gateway (CT 100):**
   - Webzugänge werden durch ein eigenes Node.js-Portal mit TOTP-Tokens geschützt.
   - Die Anbindung an das Internet erfolgt über einen ausgehenden **Cloudflare Zero Trust Tunnel** (keine offenen Port-Weiterleitungen im Heimrouter für Port 80/443 nötig).
2. **Cluster-Trust & Least-Privilege:**
   - Beide Proxmox-Nodes kommunizieren über interne SSH-Keypairs.
   - Automatisierte Skripte nutzen den lokalen **Proxmox QEMU Guest Agent** (`qm guest exec`), sodass keine Root-SSH-Verbindungen mit Passwörtern über das Netzwerk erforderlich sind.
3. **Secret Sanitization vor jedem Git-Push:**
   - Alle automatisierten Exporte filtern Passwörter, TLS-Fingerprints, Velocity-Secrets und E-Mail-Adressen via Regex heraus (`<SECRET_REDACTED>`), bevor ein Git-Commit erzeugt wird.

---

## 🔄 Automatisierung & GitOps-Sync

Ein nativer **systemd-Timer** auf PVE 1 sorgt für eine saubere, versionierte Dokumentation und Konfigurationssicherung:

- **Timer:** `homelab-git-sync.timer` (Ausführung täglich um **05:00 Uhr CEST**)
- **Skript:** `/usr/local/bin/homelab-git-sync.sh`
- **Schutzmechanismus:** Scannt den Staged Diff vor dem Commit (`git diff --staged`) nach sensiblen Zeichenfolgen und bricht bei Fund sofort ab.
- **Zugehörige GitHub-Repositories:**
  - 🔒 [`Hannes-frbg/homelab-configs`](https://github.com/Hannes-frbg/homelab-configs): Private Bereinigte Config-Backups (PVE-Jobs, Netzwerk, Velocity, Purpur)
  - 🌐 [`Hannes-frbg/homelab-architecture`](https://github.com/Hannes-frbg/homelab-architecture): Öffentliche Dokumentation & Architekturübersichten

---

## 📚 Vertiefende Dokumentation

Für vollständige Details zu jedem Konfigurationsparameter, Wiederherstellungsanleitungen (Disaster Recovery) und Netzwerkpläne:
- 📄 [Vollständige Proxmox Systemdokumentation (Markdown)](PROXMOX_SYSTEM_DOKUMENTATION.md)
- 📝 [Ausführlicher Detailbericht als Textdatei](PROXMOX_SYSTEM_DOKUMENTATION.txt)

---
*Erstellt & Gepflegt durch Antigravity DevOps Automation Agent — Stand: Oktober 2026*
