# imou-hx21-openwrt

**A field-tested journey running OpenWrt on the Imou HX21** — from flashing and package management to SQM/QoS tuning, WireGuard, NetBird, firewall hardening, diagnostics, and the troubleshooting notes that came out of actually living with this router.

<p align="left">
  <img src="https://img.shields.io/badge/OpenWrt-25.12-1D2B36?style=for-the-badge&logo=openwrt&logoColor=1DE0B1" alt="OpenWrt 25.12">
  <img src="https://img.shields.io/badge/Package%20Manager-apk-0D597F?style=for-the-badge&logo=alpinelinux&logoColor=white" alt="apk package manager">
  <img src="https://img.shields.io/badge/SoC-MT7981B%20Filogic-000000?style=for-the-badge" alt="MediaTek MT7981B Filogic">
  <br/>
  <img src="https://img.shields.io/badge/VPN-WireGuard-88171A?style=flat-square&logo=wireguard&logoColor=white" alt="WireGuard">
  <img src="https://img.shields.io/badge/VPN-NetBird-0A69DA?style=flat-square" alt="NetBird">
  <img src="https://img.shields.io/badge/QoS-CAKE%20%2F%20SQM-2C3E50?style=flat-square" alt="CAKE/SQM">
  <img src="https://img.shields.io/badge/Firewall-nftables%20(fw4)-CC0000?style=flat-square&logo=linux&logoColor=white" alt="firewall4/nftables">
  <br/>
  <img src="https://img.shields.io/badge/status-actively%20maintained-brightgreen?style=flat-square" alt="Actively maintained">
  <img src="https://img.shields.io/badge/license-MIT-blue.svg?style=flat-square" alt="MIT License">
  <img src="https://img.shields.io/github/last-commit/itachi-re/imou-hx21-openwrt?style=flat-square" alt="Last commit">
</p>

---

## Why this repo exists

The Imou HX21 (aka **Imou LC-HX3001**) is a genuinely new OpenWrt target — board support only merged upstream in late 2025. Documentation is thin and scattered across forum threads. This repo is my running record of getting it from stock firmware to a fully-tuned router: what worked, what broke, and the exact commands that fixed it — written against **OpenWrt 25.12** and the `apk` package manager it introduced.

If you're running the same board (or any recent MediaTek Filogic router) and are tired of translating 2022-era `opkg` tutorials in your head, this should save you the trouble.

---

## 📋 Table of Contents

- [Hardware at a glance](#-hardware-at-a-glance)
- [Repo structure](#-repo-structure)
- [The journey](#-the-journey)
- [Quick start](#-quick-start)
- [Documentation index](#-documentation-index)
- [Troubleshooting highlights](#-troubleshooting-highlights)
- [Known caveats](#-known-caveats)
- [Roadmap](#-roadmap)
- [Sources & further reading](#-sources--further-reading)
- [License](#license)

---

## 🔧 Hardware at a glance

| Component | Detail |
|---|---|
| SoC | MediaTek MT7981B "Filogic 820", dual-core Cortex-A53 @ ~1.3 GHz |
| RAM | 256 MB DDR3 |
| Flash | 128 MB SPI-NAND |
| Switch | MediaTek MT7531AE — 4× 10/100/1000 Mbps |
| Wi-Fi | MediaTek MT7976C, Wi-Fi 6 (AX3000-class, dual-band) |
| OpenWrt target | `mediatek/filogic` |

> 256 MB RAM and 128 MB flash are workable but not generous — every extra daemon (LuCI, multiple VPN clients, heavy logging) eats into the same budget the router needs for NAT offload and CAKE. See [`docs/complete-guide.md`](docs/complete-guide.md#hardware-overview) for what that means in practice.

---

## 📁 Repo structure

```
imou-hx21-openwrt/
├── README.md                  ← you are here
└── docs/
    ├── installation-guide.md  ← flashing OpenWrt from stock: firmware files, backup,
    │                             Linux/Windows/Android methods, recovery, UART, safety checklist
    ├── complete-guide.md      ← post-install reference: package mgmt → SQM/QoS → WireGuard →
    │                             firewall → diagnostics → backup → hardening → cheat sheet
    └── vpn-netbird.md          ← NetBird install, routing peer, exit node, DNS, troubleshooting
```

---

## 🗺️ The journey

```mermaid
flowchart LR
    A[Stock Imou\nfirmware] -->|installation-guide.md| B[Flash to\nOpenWrt 25.12]
    B --> C[apk package\nmanagement]
    C --> D[SQM / CAKE\nbufferbloat control]
    D --> E[WireGuard\nclient + server]
    E --> F[NetBird\nmesh VPN]
    F --> G[firewall4 / nftables\nhardening]
    G --> H[Diagnostics &\nbenchmarking]
    H --> I[Backup &\nrecovery drills]
    I --> J[Production-ready\nconfig profile]

    style A fill:#3a3a3a,color:#fff
    style J fill:#1DE0B1,color:#000
```

Each stage is documented with the commands actually run, what broke, and how it was diagnosed — not a clean-room tutorial.

---

## ⚡ Quick start

```sh
# Confirm you're on the apk-based image (25.12+) before copying anything from this repo
apk --version

# Refresh indexes and get LuCI + diagnostics on
apk update
apk add luci luci-ssl
/etc/init.d/uhttpd restart

# Identify real interface names before touching network/firewall config
ip -brief link show
uci show network | grep device
```

Full walkthrough: [`docs/complete-guide.md` → Quick Start](docs/complete-guide.md#quick-start)

---

## 📚 Documentation index

### [`docs/installation-guide.md`](docs/installation-guide.md) — flashing OpenWrt from stock

| Section | Covers |
|---|---|
| [Overview](docs/installation-guide.md#1-overview) | What gets replaced, reversibility, brick risk |
| [Hardware Information](docs/installation-guide.md#2-hardware-information) | Verified specs + MTD partition layout |
| [Requirements](docs/installation-guide.md#3-requirements) | Linux/Windows/Android tooling, incl. an honest look at what Android can't do |
| [Firmware Files](docs/installation-guide.md#5-firmware-files) | Exact filenames, SHA-256 checksums, download URLs for 25.12.5 |
| [Backup Stock Firmware](docs/installation-guide.md#6-backup-stock-firmware) | Dumping every MTD partition before touching anything |
| [Linux Method](docs/installation-guide.md#8-linux-method) / [Windows Method](docs/installation-guide.md#9-windows-method) | Full TFTP + SSH flashing walkthrough |
| [Recovery / Debrick](docs/installation-guide.md#15-recovery--debrick) / [UART Recovery](docs/installation-guide.md#16-uart-recovery) | Boot menu recovery paths, serial console fallback |
| [Troubleshooting](docs/installation-guide.md#17-troubleshooting) | Symptom → cause → fix, incl. common TFTP failures |
| [Safety Checklist](docs/installation-guide.md#19-safety-checklist) | Pre-flight checks before any `bl2`/`fip` write |

### [`docs/complete-guide.md`](docs/complete-guide.md) — the core post-install reference

| Section | Covers |
|---|---|
| [APK Package Management](docs/complete-guide.md#1-apk-package-management) | `opkg` → `apk` migration, full command equivalence table |
| [SQM / Smart Queue Management](docs/complete-guide.md#2-sqm--smart-queue-management) | Bufferbloat control, CAKE tuning, measuring real throughput |
| [QoS](docs/complete-guide.md#3-qos) | Traffic prioritization beyond default CAKE tins |
| [WireGuard](docs/complete-guide.md#4-wireguard) | Client and server roles, key handling, firewall zones |
| [Firewall (firewall4 / nftables)](docs/complete-guide.md#5-firewall-firewall4--nftables) | Zone/forwarding model, `fw4` internals |
| [Network Diagnostics](docs/complete-guide.md#6-network-diagnostics) | `ip`, `ubus`, `logread` workflows |
| [Performance / Benchmarking](docs/complete-guide.md#7-performance--benchmarking) | Realistic throughput ceilings for this SoC |
| [Backup / Recovery](docs/complete-guide.md#8-backup--recovery) | `sysupgrade` backup/restore, what's actually captured |
| [Recommended Configuration Profiles](docs/complete-guide.md#9-recommended-configuration-profiles) | Ready-made setups by use case |
| [Troubleshooting](docs/complete-guide.md#10-troubleshooting) | Symptom → cause → fix tables |
| [Security](docs/complete-guide.md#11-security) | Hardening checklist (LuCI/SSH exposure, key handling, least privilege) |
| [Command Cheat Sheet](docs/complete-guide.md#command-cheat-sheet) | Every command in the guide, grouped by topic |

### [`docs/vpn-netbird.md`](docs/vpn-netbird.md) — NetBird mesh VPN

| Section | Covers |
|---|---|
| [Check your OpenWrt release](docs/vpn-netbird.md#0-check-your-openwrt-release-first) | NetBird version ↔ OpenWrt release ↔ package manager matrix |
| [Install & connect](docs/vpn-netbird.md#1-install-the-netbird-package) | `apk`/`opkg` install, `netbird up`, verifying connection |
| [DNS configuration](docs/vpn-netbird.md#4-dns-configuration-optional-only-if-you-use-netbird-dns-features) | Resolving the dnsmasq/port-53 conflict |
| [Routing peer (LAN access)](docs/vpn-netbird.md#5-use-the-router-as-a-routing-peer-lan-access-from-other-netbird-peers) | Exposing the router's LAN to the mesh |
| [Exit node](docs/vpn-netbird.md#6-use-the-router-as-an-internet-exit-node-route-your-traffic-through-the-routers-wan-ip) | Routing all device traffic through the router's WAN IP |
| [Persistence](docs/vpn-netbird.md#7-persistence-across-reboots-and-firmware-upgrades) | Config survival across `sysupgrade` |
| [Troubleshooting](docs/vpn-netbird.md#8-quick-troubleshooting-reference) | Symptom → cause → fix table |

---

## 🩹 Troubleshooting highlights

A few of the sharper edges hit along the way — full detail in the linked sections:

- **`opkg` habits don't transfer.** OpenWrt 25.12 dropped `opkg` entirely for `apk`; there's no dual-mode router, and pasting `opkg install` on a 25.12 image just fails with "command not found." → [package management table](docs/complete-guide.md#opkg-vs-apk-a-quick-orientation)
- **Duplicate firewall zones.** Some VPN packages (NetBird included) auto-create their own firewall zone on install. Adding a second one manually produces an nftables `redefinition of symbol` error on `firewall restart`. Always `uci show firewall | grep -i <service>` before adding a zone. → [NetBird firewall setup](docs/vpn-netbird.md#52-create-a-firewall-zone-and-allow-forwarding)
- **SQM needs headroom, not your sync speed.** Setting SQM's download/upload to your ISP's advertised rate defeats the point — CAKE needs to be the bottleneck, not your ISP's box. → [SQM section](docs/complete-guide.md#2-sqm--smart-queue-management)
- **NetBird exit-node LAN exposure (upstream issue).** Devices routed through this router as a NetBird exit node can reach the router's *entire LAN subnet*, not just the internet, even with a scoped ACL — [netbirdio/netbird#5797](https://github.com/netbirdio/netbird/issues/5797). Mitigation and a manual verification step are documented. → [exit node caveats](docs/vpn-netbird.md#64-known-caveat--lan-exposure-through-exit-nodes)

---

## ⚠️ Known caveats

- **Board support is young.** Imou HX21 / LC-HX3001 support merged upstream in **late 2025** — treat every command in this repo as something to verify against your own firmware build (upstream OpenWrt vs. ImmortalWrt), not a guaranteed copy-paste.
- **256 MB RAM ceiling.** Running LuCI + WireGuard + NetBird + heavy logging simultaneously is workable but leaves little headroom — monitor with `free`.
- **NetBird on arm64 is "less tested"** per NetBird's own docs (x86_64 is their most-exercised platform). Watch `dmesg`/`logread` after install.
- **A bad firewall/network push over SSH can lock you out.** Every destructive command in `docs/complete-guide.md` is explicitly flagged — keep a physical-access recovery path available before editing `network` or `firewall` config remotely.

---

## 🛣️ Roadmap

- [ ] Policy-based routing (PBR) between home (TP-Link WR850N v2) and work (HX21) routers
- [ ] WireGuard site-to-site link home ↔ work
- [ ] Automated config backup → offsite sync
- [ ] banIP integration for inbound abuse mitigation
- [ ] Writeup comparing NetBird vs. Tailscale on the same hardware

---

## 📖 Sources & further reading

Full citation list lives in [`docs/complete-guide.md` → Sources](docs/complete-guide.md#sources), covering the OpenWrt 25.12 release notes, the apk transition, CAKE/SQM references, firewall4 internals, and the Imou HX21 board-support merge threads. NetBird-specific sources are in [`docs/vpn-netbird.md` → Sources](docs/vpn-netbird.md#sources).

---

## License

MIT — see [`LICENSE`](LICENSE). Documentation content reflects a specific point-in-time OpenWrt/NetBird release; verify version numbers against your own hardware before relying on it in production.

---

<p align="center"><sub>Maintained by <a href="https://github.com/itachi-re">@itachi-re</a> · Built on OpenWrt 25.12 · MT7981B Filogic</sub></p>
