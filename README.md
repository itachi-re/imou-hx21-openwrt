# imou-hx21-openwrt

**A field-tested journey running OpenWrt on the Imou HX21** — from flashing and package management to SQM/QoS tuning, WireGuard, NetBird, firewall hardening, diagnostics, and the troubleshooting notes that came out of actually living with this router.

<p align="left">
  <img src="https://img.shields.io/badge/OpenWrt-25.12-1D2B36?style=for-the-badge&logo=openwrt&logoColor=1DE0B1" alt="OpenWrt 25.12">
  <img src="https://img.shields.io/badge/Package%20Manager-apk-0D597F?style=for-the-badge&logo=alpinelinux&logoColor=white" alt="apk package manager">
  <img src="https://img.shields.io/badge/SoC-MT7981B%20Filogic-000000?style=for-the-badge" alt="MediaTek MT7981B Filogic">
  <br/>
  <img src="https://img.shields.io/badge/VPN-WireGuard-88171A?style=flat-square&logo=wireguard&logoColor=white" alt="WireGuard">
  <img src="https://img.shields.io/badge/VPN-NetBird-0A69DA?style=flat-square" alt="NetBird">
  <img src="https://img.shields.io/badge/VPN-Tailscale-101828?style=flat-square&logo=tailscale&logoColor=white" alt="Tailscale">
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
| Wi-Fi | Wi-Fi 6 (AX3000-class, dual-band) — driver reports `embedded [MediaTek MT7981]` on both radios, not a separate MT7976C part¹ |
| OpenWrt target | `mediatek/filogic` |

> ¹ Earlier hardware notes assumed a MT7976C companion radio based on typical MT7981B reference
> designs. Two independent checks on this unit — `dmesg | grep -i mt7976` (empty) and `iwinfo`
> (`Hardware: embedded [MediaTek MT7981]` on both `phy0` and `phy1`) — don't confirm that part
> number and point toward the WiFi being handled by the SoC's own integrated WMAC instead. Kept
> as an open question rather than a confirmed correction — see [`flashing-field-notes.md` §5.1](docs/flashing-field-notes.md#51-verify-both-radios-came-up).

> 256 MB RAM and 128 MB flash are workable but not generous — every extra daemon (LuCI, multiple VPN clients, heavy logging) eats into the same budget the router needs for NAT offload and CAKE. See [`docs/complete-guide.md`](docs/complete-guide.md#hardware-overview) for what that means in practice.

---

## 📁 Repo structure

```
imou-hx21-openwrt/
├── README.md                    ← you are here
└── docs/
    ├── installation-guide.md    ← flashing OpenWrt from stock: firmware files, backup,
    │                               Linux/Windows/Android methods, recovery, UART, safety checklist
    ├── complete-guide.md        ← post-install reference: package mgmt → SQM/QoS → WireGuard →
    │                               firewall → diagnostics → backup → hardening → cheat sheet
    ├── vpn-netbird.md           ← NetBird install, routing peer, exit node, DNS, troubleshooting
    ├── vpn-tailscale.md         ← Tailscale install, subnet router, exit node, Tailscale SSH,
    │                               ACL overview, firewall4/nftables gotchas, removal
    ├── migrating-to-netbird.md  ← full NetBird install/config walkthrough, including a clean
    │                               Tailscale-removal section for anyone moving off it entirely
    └── flashing-field-notes.md  ← real backup/flash/first-boot session log: MTD partition dumps,
                                    in-place `mtd write` flashing from stock SSH (no TFTP), the
                                    post-flash IP change, opkg→apk mixup, wireless bring-up
```

> ⚠️ **Overlap note:** `migrating-to-netbird.md` is a self-contained NetBird guide (install →
> routing peer → exit node → troubleshooting) with a Tailscale-removal section bolted onto the
> front — it isn't a diff against `vpn-netbird.md`. Until these two get reconciled into one
> doc, treat `migrating-to-netbird.md` as the one to follow if you're coming *from* Tailscale,
> and `vpn-netbird.md` for a from-scratch NetBird install.
>
> **`flashing-field-notes.md` vs. `installation-guide.md`:** the installation guide is the
> prescriptive, start-to-finish walkthrough (TFTP-based). The field notes document a *different*,
> real session using an alternate method — in-place `mtd write` flashing over SSH from stock
> firmware, no TFTP involved — kept as a chronological field report rather than folded into the
> main guide. Cross-reference both if you're choosing a flashing method.

---

## 🗺️ The journey

```mermaid
flowchart LR
    A[Stock Imou\nfirmware] -->|installation-guide.md| B[Flash to\nOpenWrt 25.12]
    B --> C[apk package\nmanagement]
    C --> D[SQM / CAKE\nbufferbloat control]
    D --> E[WireGuard\nclient + server]
    E --> F1[Tailscale\nmesh VPN]
    F1 -->|migrating-to-netbird.md| F[NetBird\nmesh VPN]
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

### [`docs/flashing-field-notes.md`](docs/flashing-field-notes.md) — real backup/flash session, alternate method

| Section | Covers |
|---|---|
| [Stock firmware has open root SSH](docs/flashing-field-notes.md#0-starting-point-stock-firmware-already-has-open-root-ssh) | No unlock file needed on this firmware revision — verify on your own unit before relying on it |
| [Back up stock partitions](docs/flashing-field-notes.md#1-back-up-stock-partitions-before-touching-anything) | MTD layout, `dd` + `scp -O` workflow, the nested-SSH chunking trap, `md5sum` verification |
| [Flash via `mtd write`](docs/flashing-field-notes.md#2-flashing-openwrt-in-place-via-mtd-write-from-stock-firmware) | In-place FIP flash from a live stock-firmware SSH session — no TFTP required |
| [Post-flash IP change](docs/flashing-field-notes.md#24-the-routers-lan-ip-changes-after-this-flash) | Router moves from `192.168.10.1` (stock) to `192.168.1.1` (OpenWrt default) |
| [Recovering host connectivity](docs/flashing-field-notes.md#3-recovering-host-network-connectivity-after-the-reboot) | DHCP race after reboot, static-IP workaround |
| [First boot](docs/flashing-field-notes.md#4-first-boot-on-openwrt) | Confirming the build, `opkg` vs `apk`, temporary manual route for `apk update` before WAN is set up |
| [Wireless configuration](docs/flashing-field-notes.md#5-wireless-configuration) | `uci` SSID/radio setup, `default_radio1` disabled by default, country code, DFS-avoidance channel note |
| [Troubleshooting](docs/flashing-field-notes.md#7-quick-troubleshooting-reference) | Symptom → cause → fix table |

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

### [`docs/vpn-tailscale.md`](docs/vpn-tailscale.md) — Tailscale on OpenWrt

| Section | Covers |
|---|---|
| [Check release & package manager](docs/vpn-tailscale.md#0-check-your-openwrt-release-and-package-manager-first) | `opkg` vs `apk` by OpenWrt version, flash-space requirements |
| [Install the package](docs/vpn-tailscale.md#1-install-the-package) | Official feed package, `kmod-tun` dependency, optional LuCI front-ends |
| [`ip6tables` gap on firewall4](docs/vpn-tailscale.md#3-known-gap-on-firewall4nftables-builds-missing-ip6tables) | Same `iptables-nft`/`ip6tables-nft` shim fix NetBird needs |
| [Authenticate the router](docs/vpn-tailscale.md#4-authenticate-the-router) | Interactive browser login vs. non-interactive pre-auth key |
| [Firewall integration](docs/vpn-tailscale.md#5-firewall-integration) | When a manual zone is/isn't needed, `--netfilter-mode`, anonymous-UCI-section deletion gotcha |
| [DNS & MagicDNS](docs/vpn-tailscale.md#6-dns-and-magicdns) | Quad100 resolver, `--accept-dns`, forwarding `*.ts.net` via dnsmasq |
| [Subnet router](docs/vpn-tailscale.md#7-subnet-router-advertise-your-lan-to-the-tailnet) | Advertising the LAN, admin-console route approval, `--accept-routes` |
| [Exit node](docs/vpn-tailscale.md#8-exit-node-route-all-internet-traffic-through-the-router) | Advertise/approve/use, permission model, SNAT + subnet-router caveat |
| [Tailscale SSH](docs/vpn-tailscale.md#9-tailscale-ssh-optional) | Tailnet-identity SSH, ACL-permission risk of locking yourself out |
| [Removing Tailscale](docs/vpn-tailscale.md#12-removing-tailscale) | Points to the full removal walkthrough in `migrating-to-netbird.md` |
| [Troubleshooting](docs/vpn-tailscale.md#13-quick-troubleshooting-reference) | Symptom → cause → fix table |

### [`docs/migrating-to-netbird.md`](docs/migrating-to-netbird.md) — NetBird install, with a clean break from Tailscale

| Section | Covers |
|---|---|
| [Check release & NetBird version](docs/migrating-to-netbird.md#0-check-your-openwrt-release-first) | `opkg`/`apk` and NetBird version by OpenWrt release, setup-key generation |
| [Remove Tailscale completely first](docs/migrating-to-netbird.md#1-migrating-from-tailscale-remove-it-completely-first) | Clean logout, package purge, leftover state files, anonymous UCI zone/forwarding cleanup by index |
| [Install the NetBird package](docs/migrating-to-netbird.md#2-install-the-netbird-package) | `opkg`/`apk` install across releases |
| [`ip6tables` gap on firewall4](docs/migrating-to-netbird.md#32-known-gap-on-firewall4nftables-builds-missing-ip6tables) | The `wt0` interface never appearing until `iptables-nft`/`ip6tables-nft` are installed |
| [Connect to your NetBird network](docs/migrating-to-netbird.md#4-connect-the-router-to-your-netbird-network) | Cloud vs. self-hosted, `Idle`/lazy-connection false alarms, relayed vs. P2P |
| [DNS configuration](docs/migrating-to-netbird.md#5-dns-configuration-optional-only-if-you-use-netbird-dns-features) | Pinning NetBird's resolver off port 53, dnsmasq forwarding |
| [Routing peer (LAN access)](docs/migrating-to-netbird.md#6-use-the-router-as-a-routing-peer-lan-access-from-other-netbird-peers) | Registering `wt0`, firewall zone, network-resource + ACL policy |
| [Exit node](docs/migrating-to-netbird.md#7-use-the-router-as-an-internet-exit-node-route-your-traffic-through-the-routers-wan-ip) | Dashboard setup, Auto Apply, verifying via `ifconfig.me` |
| [Known caveat — LAN exposure via exit nodes](docs/migrating-to-netbird.md#74-known-caveat--lan-exposure-through-exit-nodes) | netbirdio/netbird#5797, access-control-group mitigation, manual verification |
| [Persistence](docs/migrating-to-netbird.md#8-persistence-across-reboots-and-firmware-upgrades) | Config path by OpenWrt version, confirmed survives reboot |
| [Troubleshooting](docs/migrating-to-netbird.md#9-quick-troubleshooting-reference) | Symptom → cause → fix table, incl. Tailscale-removal-specific issues |

---

## 🩹 Troubleshooting highlights

A few of the sharper edges hit along the way — full detail in the linked sections:

- **`opkg` habits don't transfer.** OpenWrt 25.12 dropped `opkg` entirely for `apk`; there's no dual-mode router, and pasting `opkg install` on a 25.12 image just fails with "command not found." → [package management table](docs/complete-guide.md#opkg-vs-apk-a-quick-orientation)
- **Duplicate firewall zones.** Some VPN packages (NetBird included) auto-create their own firewall zone on install. Adding a second one manually produces an nftables `redefinition of symbol` error on `firewall restart`. Always `uci show firewall | grep -i <service>` before adding a zone. → [NetBird firewall setup](docs/vpn-netbird.md#52-create-a-firewall-zone-and-allow-forwarding)
- **SQM needs headroom, not your sync speed.** Setting SQM's download/upload to your ISP's advertised rate defeats the point — CAKE needs to be the bottleneck, not your ISP's box. → [SQM section](docs/complete-guide.md#2-sqm--smart-queue-management)
- **NetBird exit-node LAN exposure (upstream issue).** Devices routed through this router as a NetBird exit node can reach the router's *entire LAN subnet*, not just the internet, even with a scoped ACL — [netbirdio/netbird#5797](https://github.com/netbirdio/netbird/issues/5797). Mitigation and a manual verification step are documented. → [exit node caveats](docs/migrating-to-netbird.md#74-known-caveat--lan-exposure-through-exit-nodes)
- **`ip6tables` missing on firewall4/nftables builds — hits both NetBird and Tailscale identically.** Modern OpenWrt ships `nft` but not the legacy `iptables`/`ip6tables` shims that both daemons' firewall managers shell out to directly. Symptom: `wt0`/`tailscale0` never comes up, or the daemon errors on `status`/`down`. Fix once (`apk add iptables-nft ip6tables-nft`), and it covers both VPNs. → [NetBird](docs/migrating-to-netbird.md#32-known-gap-on-firewall4nftables-builds-missing-ip6tables) / [Tailscale](docs/vpn-tailscale.md#3-known-gap-on-firewall4nftables-builds-missing-ip6tables)
- **Anonymous UCI firewall sections can't be deleted by name.** Zones/forwarding rules added via `uci add firewall zone` are anonymous — `uci delete firewall.tailscale_zone` (or `.netbird_zone`) fails with `Entry not found` even though the zone clearly exists. Find the real index first (`uci show firewall | grep -i <service>`) and delete by `@zone[N]`/`@forwarding[N]`, highest index first. → [Tailscale removal](docs/migrating-to-netbird.md#1-migrating-from-tailscale-remove-it-completely-first)
- **Removing Tailscale for NetBird isn't just `apk del tailscale`.** The daemon deliberately leaves node-identity state behind (`/etc/config/tailscaled.state`, `/etc/tailscale/`) so a reinstall doesn't burn a new key — if you're leaving for good, that has to be wiped explicitly, and any LuCI front-end has to come off first or `apk del` refuses. → [full removal walkthrough](docs/migrating-to-netbird.md#1-migrating-from-tailscale-remove-it-completely-first)
- **`scp` to/from the router fails with `sftp-server: not found`.** Stock and early OpenWrt builds' BusyBox `ash` has no SFTP subsystem, but modern `scp` defaults to SFTP. Force the legacy protocol with `scp -O` in both directions. → [backup workflow](docs/flashing-field-notes.md#13-gotcha-scp-pull-from-the-router-fails-outright)
- **Never nest an `ssh ... dd` loop inside an already-open SSH session to the same router.** Doing so spawns a nested connection back to itself and stalls on a host-key prompt. Run per-chunk backup loops from your host machine's own terminal. → [MTD dump walkthrough](docs/flashing-field-notes.md#14-dumping-the-1145mb-ubi-partition)
- **The router's LAN IP changes after an in-place flash.** Stock firmware answers at `192.168.10.1`; after flashing OpenWrt and rebooting, it comes back at the default `192.168.1.1` — a different subnet. `ssh root@192.168.10.1` hanging right after a flash usually just means the router moved, not that something broke. → [IP change note](docs/flashing-field-notes.md#24-the-routers-lan-ip-changes-after-this-flash)

---

## ⚠️ Known caveats

- **Board support is young.** Imou HX21 / LC-HX3001 support merged upstream in **late 2025** — treat every command in this repo as something to verify against your own firmware build (upstream OpenWrt vs. ImmortalWrt), not a guaranteed copy-paste.
- **256 MB RAM ceiling.** Running LuCI + WireGuard + NetBird/Tailscale + heavy logging simultaneously is workable but leaves little headroom — monitor with `free`. Tailscale alone needs roughly 20–25 MB of free flash for the official package; very low-flash devices may not have room for it.
- **NetBird on arm64 is "less tested"** per NetBird's own docs (x86_64 is their most-exercised platform). Watch `dmesg`/`logread` after install. In practice on this HX21, the only real gap found was the shared `ip6tables-nft` dependency, not anything MT7981B/arm64-specific.
- **Tailscale's free tier has limits** (3 users / 100 devices / 1 subnet router at time of writing) — check current limits in your account before planning around them, they change.
- **Running a device as both a Tailscale subnet router and exit node with `--snat-subnet-routes=false` can drop upstream traffic**, per Tailscale's own docs — split the two roles across devices if you need both without SNAT.
- **A bad firewall/network push over SSH can lock you out.** Every destructive command in `docs/complete-guide.md` is explicitly flagged — keep a physical-access recovery path available before editing `network` or `firewall` config remotely. The same caution applies to enabling Tailscale SSH as your only remote-access path: confirm your ACL policy actually grants it before relying on it, ideally with console access as a fallback while testing.
- **Stock firmware (at least on this unit) ships with unauthenticated root SSH.** No password, no unlock file needed — SSH just drops into a root shell. Set a password immediately on first access (`passwd`), before the device is anywhere near a real LAN or WAN. This may be firmware-revision-specific; verify on your own unit rather than assuming. → [details](docs/flashing-field-notes.md#0-starting-point-stock-firmware-already-has-open-root-ssh)
- **`5GHz` radio ships disabled by default.** `wireless.default_radio1.disabled` defaults to `'1'` — a `uci commit` that only sets SSID/key on radio1 will succeed with no error while the radio stays off. Explicitly set `disabled='0'`. → [wireless setup](docs/flashing-field-notes.md#5-wireless-configuration)

---

## 🛣️ Roadmap

- [x] NetBird mesh VPN — install, routing peer, exit node ([`docs/vpn-netbird.md`](docs/vpn-netbird.md))
- [x] Tailscale — install, subnet router, exit node, Tailscale SSH ([`docs/vpn-tailscale.md`](docs/vpn-tailscale.md))
- [x] Tailscale → NetBird migration path, incl. full Tailscale removal ([`docs/migrating-to-netbird.md`](docs/migrating-to-netbird.md))
- [x] Real-world backup/flash/first-boot session log, alternate `mtd write` flashing method ([`docs/flashing-field-notes.md`](docs/flashing-field-notes.md))
- [ ] Confirm the actual 5GHz companion-radio part number (MT7976C vs. SoC-integrated) — current evidence points against MT7976C, not confirmed either way
- [ ] Reconcile `vpn-netbird.md` and `migrating-to-netbird.md` into a single canonical NetBird doc
- [ ] Policy-based routing (PBR) between home (TP-Link WR850N v2) and work (HX21) routers
- [ ] WireGuard site-to-site link home ↔ work
- [ ] Automated config backup → offsite sync
- [ ] banIP integration for inbound abuse mitigation
- [ ] Writeup comparing NetBird vs. Tailscale on the same hardware — now that both docs exist, this is mostly a synthesis pass

---

## 📖 Sources & further reading

Full citation list lives in [`docs/complete-guide.md` → Sources](docs/complete-guide.md#sources), covering the OpenWrt 25.12 release notes, the apk transition, CAKE/SQM references, firewall4 internals, and the Imou HX21 board-support merge threads. VPN-specific sources are in [`docs/vpn-netbird.md` → Sources](docs/vpn-netbird.md#sources), [`docs/vpn-tailscale.md` → Sources](docs/vpn-tailscale.md#sources), and [`docs/migrating-to-netbird.md` → Sources](docs/migrating-to-netbird.md#sources).

---

## License

MIT — see [`LICENSE`](LICENSE). Documentation content reflects a specific point-in-time OpenWrt/NetBird release; verify version numbers against your own hardware before relying on it in production.

---

<p align="center"><sub>Maintained by <a href="https://github.com/itachi-re">@itachi-re</a> · Built on OpenWrt 25.12 · MT7981B Filogic</sub></p>
