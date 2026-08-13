# Imou HX21 on OpenWrt — Practical Reference Guide

A field guide to running OpenWrt on the **Imou HX21** (aka **Imou LC-HX3001**), covering package management, SQM/bufferbloat control, QoS, WireGuard (client + server), the firewall4/nftables stack, diagnostics, benchmarking, backup/recovery, and hardened configuration profiles.

> **Target release:** OpenWrt **25.12** (currently 25.12.x, the actively maintained stable series as of writing). OpenWrt 25.12 replaced the `opkg` package manager with **`apk`** (Alpine Package Keeper). Where behavior differs from OpenWrt 24.10 or older, it is called out explicitly in a callout box.
>
> **Read this first:**
> - OpenWrt 24.10 is the *previous* stable series and still uses `opkg`. It is scheduled to lose security support around September 2026. If your HX21 is still running 24.10, see the [opkg vs apk note](#opkg-vs-apk-a-quick-orientation) before copying any command from this guide verbatim.
> - Interface names (`eth0`, `wan`, `br-lan`, etc.), exact package availability, and default zone names can vary between firmware images, ImmortalWrt vs. upstream OpenWrt builds, and how your HX21 was flashed. **Always confirm interface/device names on your own router** (`ip addr`, `Network → Interfaces` in LuCI) before pasting commands.
> - The HX21 is a genuinely new (2025) port. Community support (ImmortalWrt "imou_lc-hx3001" target, and the upstream `mediatek: add support for Imou HX21` commit merged into OpenWrt) is comparatively young — double-check the forum threads linked in [Sources](#sources) for any regressions in your specific firmware build before doing a production deployment.

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Hardware Overview](#hardware-overview)
3. [Quick Start](#quick-start)
4. [APK Package Management](#1-apk-package-management)
5. [SQM / Smart Queue Management](#2-sqm--smart-queue-management)
6. [QoS](#3-qos)
7. [WireGuard](#4-wireguard)
8. [Firewall (firewall4 / nftables)](#5-firewall-firewall4--nftables)
9. [Network Diagnostics](#6-network-diagnostics)
10. [Performance / Benchmarking](#7-performance--benchmarking)
11. [Backup / Recovery](#8-backup--recovery)
12. [Recommended Configuration Profiles](#9-recommended-configuration-profiles)
13. [Troubleshooting](#10-troubleshooting)
14. [Security](#11-security)
15. [Command Cheat Sheet](#command-cheat-sheet)
16. [Sources](#sources)

---

## Prerequisites

- An Imou HX21 (Imou LC-HX3001 global variant) already flashed to OpenWrt or ImmortalWrt. Flashing stock firmware → OpenWrt is **out of scope** for this document; see the forum threads in [Sources](#sources) for the current flashing procedure, since it depends on your OEM firmware version and involves opening the SSH-access window on the stock web UI.
- SSH access to the router (`ssh root@192.168.1.1` or your router's LAN IP) and/or LuCI web access.
- Basic comfort with the command line. Every destructive or connectivity-affecting command below is explicitly marked.
- A known-working recovery path (serial/UART console, or at minimum physical access to hold the reset button) before you touch firewall or network configuration remotely. **A bad firewall or network config change delivered over SSH/LuCI can lock you out of the router.**

> ⚠️ **Warning:** Several commands in this guide reload the network stack or the firewall. If you are connected over Wi-Fi or through the interface you're modifying, you can lose your session. Prefer a wired LAN connection while editing `network` or `firewall` config, and keep a fallback plan (physical access, failsafe boot, or a scheduled revert) ready.

---

## Hardware Overview

| Component | Detail |
|---|---|
| SoC | MediaTek MT7981B "Filogic 820", dual-core Cortex-A53 @ ~1.3 GHz |
| RAM | 256 MB DDR3 |
| Flash | 128 MB SPI-NAND |
| Switch | MediaTek MT7531AE, 4× 10/100/1000 Mbps Ethernet |
| Wi-Fi | MediaTek MT7976C, Wi-Fi 6 (AX3000-class, dual-band) |
| Power | DC 12V / 1A |
| OpenWrt target | `mediatek/filogic` (shared with the Imou LC-HX3001 board definition) |

**What this means practically:**

- **CPU:** A dual-core 1.3 GHz Cortex-A53 is modest by desktop standards but is a fairly capable *router* CPU — it has a working NAT hardware/software flow offload path via the MT7531 switch and MediaTek's Filogic offload engine, which most Filogic-class boxes rely on to hit near-gigabit NAT throughput. Software-only packet processing (SQM, CAKE, WireGuard encryption) draws directly on this same CPU budget, and single-core-bound work (like a single CAKE queue) will not fully use both cores.
- **RAM:** 256 MB is enough for routing, SQM, a handful of WireGuard tunnels, and a modest firewall ruleset, but it is not a lot of headroom if you install a full LuCI + multiple VPN daemons + heavy logging. Keep an eye on `free` after adding packages.
- **Flash:** 128 MB is comfortable for OpenWrt's default image plus a reasonable set of extra packages, but avoid installing large package sets you don't need (e.g., full Python, multiple redundant VPN implementations).
- **Gigabit Ethernet, no multi-gig WAN.** Your realistic throughput ceiling for any CPU-processed feature (SQM, WireGuard) is bounded well under 1 Gbps — see [SQM](#2-sqm--smart-queue-management) and [WireGuard](#4-wireguard) for concrete numbers and how to determine yours.

---

## Quick Start

A minimal path to a usable, up-to-date router:

```sh
# 1. Confirm the package manager in use (informational)
apk --version

# 2. Refresh package indexes
apk update

# 3. Install LuCI (if not already present) and basic diagnostics
apk add luci luci-ssl

# 4. Reboot LuCI's web server if you just installed it
/etc/init.d/uhttpd restart

# 5. Identify your real interface names before configuring anything
ip -brief link show
uci show network | grep device
```

From here, jump to the section covering what you actually want to do: [SQM](#2-sqm--smart-queue-management) for bufferbloat, [WireGuard](#4-wireguard) for VPN, or [Recommended Configuration Profiles](#9-recommended-configuration-profiles) for a ready-made setup.

---

## 1. APK Package Management

### What `apk` is in OpenWrt 25.12+

Starting with OpenWrt 25.12, **`apk` (Alpine Package Keeper)** — the package manager from Alpine Linux — replaced `opkg` as OpenWrt's default package manager. The OpenWrt-maintained fork of `opkg` had become unmaintained upstream, so the project adopted `apk`, which is actively developed and battle-tested in Alpine Linux. Most package *names* are unchanged; the command-line syntax is different. OpenWrt publishes an official opkg→apk cheat sheet for anyone migrating existing habits or scripts.

> **24.10 vs 25.12+:** OpenWrt 24.10 and earlier use `opkg` exclusively. OpenWrt 25.12 and newer use `apk` exclusively. There is no dual-mode router — check which one you have with `apk --version` (succeeds on 25.12+) or `opkg --version` (succeeds on 24.10 and earlier).

### opkg vs apk: a quick orientation

| Task | opkg (24.10 and older) | apk (25.12+) |
|---|---|---|
| Refresh package lists | `opkg update` | `apk update` |
| Install a package | `opkg install <pkg>` | `apk add <pkg>` |
| Remove a package | `opkg remove <pkg>` | `apk del <pkg>` |
| Search for a package | `opkg list \| grep <term>` | `apk search <term>` |
| List available packages | `opkg list` | `apk list` |
| List installed packages | `opkg list-installed` | `apk list -I` (or `apk list --installed`) |
| List upgradable packages | `opkg list-upgradable` | `apk list -u` (or `apk list --upgradeable`) |
| Show package info | `opkg info <pkg>` | `apk info <pkg>` |
| List files owned by a package | `opkg files <pkg>` | `apk info -L <pkg>` |
| Force reinstall / ignore cache | `opkg install --force-reinstall` | `apk add -U <pkg>` (see below) |

Do **not** copy `opkg` commands into a 25.12+ shell "because they look similar" — `opkg` will simply not exist as a binary on an apk-based image, and the flag semantics differ where both exist.

### Core `apk` commands

```sh
# Refresh the local package index against configured repositories
apk update

# Install a package (pulls dependencies automatically)
apk add sqm-scripts

# Remove a package
apk del sqm-scripts

# Search the index for a package name/description match
apk search wireguard

# List every package available in the configured repos
apk list

# List only installed packages
apk list -I

# List packages that have an upgrade available
apk list -u

# Force apk to re-resolve against the latest index before installing
# (useful when you just changed a repo URL or added a feed)
apk -U add <package>
```

`apk -U` (or `apk update` immediately before `apk add`) is the equivalent of "don't trust my possibly-stale local package index, go re-check the repo first." Use it after editing `/etc/apk/repositories` or when a package "isn't found" but you know it exists.

### Inspecting repositories and how OpenWrt package feeds work

OpenWrt's package feeds are defined per-target and per-release. On an apk-based system, repository URLs live in `/etc/apk/repositories`:

```sh
cat /etc/apk/repositories
```

You'll typically see entries scoped to your specific **release** and **target/subtarget** (for the HX21 this is the `mediatek/filogic` target), for example a base feed, a `packages` feed (extra community packages), and a `luci` feed (LuCI web-UI components). This is why package availability is not identical across OpenWrt devices: a package has to be built for your specific CPU architecture (`aarch64_cortex-a53` for the HX21) and published to that target's feed. A package that exists for `x86_64` does not automatically exist for the HX21.

To confirm what architecture/target your build expects:

```sh
cat /etc/openwrt_release
ubus call system board | grep -E 'target|release'
```

### Determining the correct package for your release/target

1. Check `/etc/openwrt_release` for your exact version string (e.g. `25.12.x`) and target (`mediatek/filogic`).
2. Use `apk search <term>` on the router itself — this only shows packages actually available to *your* device's configured feeds, which is more reliable than browsing a generic package list online.
3. If you want to browse feeds from a PC first, use the OpenWrt package/firmware selector for your exact release and target before assuming a package exists for the HX21.

### Installing LuCI packages

LuCI itself is a metapackage plus per-feature app/proto packages:

```sh
apk update
apk add luci                 # Core LuCI web UI
apk add luci-ssl              # HTTPS support for the LuCI web server
apk add luci-app-sqm          # LuCI page for SQM (after installing sqm-scripts)
apk add luci-proto-wireguard  # WireGuard support inside "Network > Interfaces"
apk add luci-app-wireguard    # WireGuard status/overview LuCI page
```

After installing a new LuCI app package, restart `uhttpd` (LuCI's web server) so it picks up the new menu entries:

```sh
/etc/init.d/uhttpd restart
```

### Troubleshooting package installation failures

| Symptom | Likely cause | What to check |
|---|---|---|
| `ERROR: unable to select packages` / "no such package" | Package doesn't exist for your target, or index is stale | `apk update` then retry; `apk search <name>` to confirm it exists for your target |
| `apk: bad signature` / verification error | Corrupted download, clock badly wrong, or tampered feed | Check `date`, re-run `apk update`, verify `/etc/apk/repositories` points at official/trusted URLs |
| Install succeeds but LuCI page missing | uhttpd not restarted, or browser cache | `/etc/init.d/uhttpd restart`, hard-refresh browser |
| "not enough space" | Flash nearly full (128 MB total is not huge) | `df -h /overlay`, remove unneeded packages, avoid duplicate VPN stacks |
| Package installs but service doesn't start | Missing `/etc/config/<service>` default config, or service not enabled | `/etc/init.d/<service> enable && /etc/init.d/<service> start`; check `logread` |

Useful diagnostic sequence when a package fails:

```sh
apk update                 # make sure the index isn't stale
apk search <partial-name>  # confirm exact package name
df -h /overlay             # confirm you have flash space
apk add <exact-name>
logread | tail -n 50       # check for service-level errors after install
```

### Why blindly running `apk upgrade` is discouraged on OpenWrt

Unlike a general-purpose Linux distribution, OpenWrt's package set is tightly coupled to the *specific firmware build* it shipped with — kernel version, kernel module ABI, and base libraries are all pinned together. Running a broad `apk upgrade` can:

- Pull in a kernel module package that no longer matches your running kernel, breaking Wi-Fi, switch, or hardware-offload drivers.
- Exceed your available flash/RAM if newer package versions are larger.
- Leave the router in a partially-upgraded, unsupported state that's hard to reproduce or get help for on the forums.

**What to do instead when you want to upgrade OpenWrt:** perform a full firmware **sysupgrade** to a newer release/build (ideally via **Attended Sysupgrade**, which is included by default in LuCI as of 25.12 and rebuilds a firmware image that already contains your currently-installed extra packages), rather than patch packages in place. See [Backup / Recovery](#8-backup--recovery) for the safe upgrade procedure.

If you only need a single security-fix package updated (not a whole-system jump), it's reasonable to update that one package specifically (`apk update && apk add --upgrade <package>`), but treat that as an exception, not routine maintenance.

---

## 2. SQM / Smart Queue Management

### What bufferbloat is

Bufferbloat is excessive, uncontrolled queuing delay that appears when a network link (very often the WAN uplink, but also Wi-Fi and switch ports) has oversized buffers. When those buffers fill up under load, latency-sensitive traffic (VoIP, video calls, gaming, even DNS lookups) can queue for hundreds of milliseconds to seconds behind a large download or upload, even though the connection isn't "full" in the throughput sense.

### What SQM does, and why it improves latency under load

SQM ("Smart Queue Management") is OpenWrt's practical implementation of active queue management (AQM) plus rate shaping. It:

1. Shapes traffic to a rate slightly **below** the true bottleneck rate of your WAN link, so the router's own queue — not your ISP's oversized modem/CPE buffer — becomes the place where congestion is managed.
2. Applies an AQM algorithm (fq_codel or CAKE) to that shaped queue, which keeps the queue short and fairly shares bandwidth between flows, dropping/marking packets early enough to signal TCP to back off before latency explodes.

The result: your maximum throughput drops slightly (because you're shaping under line rate), but latency under load — the thing that actually ruins video calls and gaming — improves dramatically, often from many hundreds of milliseconds down to tens of milliseconds.

### CAKE vs fq_codel

- **fq_codel** — Fair-queuing + CoDel AQM. Lower CPU cost, good general-purpose behavior, but simpler feature set (no built-in shaper, no traffic-priority tin structure).
- **CAKE** ("Common Applications Kept Enhanced") — A newer, more integrated design that combines shaping, fair queuing, and AQM in a single qdisc, adds host-fairness, built-in DSCP-aware priority tins, and link-layer overhead compensation (ATM/PPPoE/Ethernet framing accounting) without extra scripting. CAKE is generally the recommended default on OpenWrt today and is what `sqm-scripts`/LuCI SQM defaults to.
- **Trade-off on the HX21:** CAKE does more work per packet than fq_codel (it maintains more internal state for its host-fairness and tin logic), so it costs more CPU. On a dual-core 1.3 GHz Cortex-A53 this is usually still very manageable at typical home-broadband rates (tens to a few hundred Mbps), but if you are shaping close to your CPU's ceiling and see high shaping-thread CPU usage, `fq_codel` is a valid fallback with lower overhead.

> As of the Linux/OpenWrt versions aligned with 25.12, a multi-queue variant, **CAKE_MQ**, exists that can spread CAKE's work across multiple CPU cores. Whether it's available and beneficial depends on your specific firmware build and the SoC's queue/IRQ layout — treat this as something to verify on your own HX21 build (`tc qdisc` and kernel module list) rather than something guaranteed to be present.

### SQM vs traditional QoS

SQM primarily solves *latency under load* through active queue management and rate shaping; it does very little manual "put video calls first" classification beyond CAKE's automatic DSCP-aware tins. Traditional QoS (see [section 3](#3-qos)) is about *explicit prioritization* between traffic classes. They are complementary, not competing — see [QoS vs SQM](#difference-between-qos-and-sqm) below.

### How SQM works internally (high level)

1. An **egress (upload)** qdisc is attached directly to your WAN-facing interface, shaping outbound traffic to your configured upload rate.
2. For **ingress (download)**, Linux can't natively shape traffic before it arrives, so SQM creates a virtual **IFB (Intermediate Functional Block)** device, redirects incoming WAN traffic through it, and shapes/queues it there — effectively "shaping after the fact but before it reaches your LAN," which still controls queueing delay because the bottleneck for download bufferbloat is usually *upstream* in the ISP's equipment, and shaping slightly under that rate prevents the upstream buffer from filling in the first place.
3. CAKE (or fq_codel) then manages that shaped queue: multiple flows are queued fairly, dequeued round-robin-ish, and CoDel-style dropping keeps queue occupancy — and thus delay — bounded.

### CPU performance considerations on the Imou HX21

- The HX21's Cortex-A53 cores are adequate for CAKE/fq_codel at typical residential rates (up to a few hundred Mbps is commonly reported as fine on similar Filogic-class hardware), but SQM is inherently CPU-bound software work, so:
  - Expect a real throughput ceiling under SQM+CAKE that is measurably below your raw NAT-only (offloaded) throughput.
  - If your WAN is close to or above ~500 Mbps–1 Gbps, budget for testing — you may need to shape more conservatively than "just under line rate" to leave CPU headroom, or accept a lower CAKE-shaped ceiling than your raw connection speed.
  - **Hardware/software flow offloading and SQM are mutually exclusive in practice** — see the dedicated warning below.

### Required packages for current OpenWrt (apk names)

```sh
apk update
apk add sqm-scripts        # Core SQM shaping/AQM scripts
apk add luci-app-sqm       # LuCI "SQM QoS" configuration page (optional but recommended)
```

> Package name is consistent across recent OpenWrt releases; always confirm with `apk search sqm` on your router since exact naming can shift between releases.

### LuCI configuration

1. **Network → SQM QoS**
2. Add/enable an SQM instance on your real WAN interface (confirm the name — see next subsection).
3. Set **Download** and **Upload** speeds (in kbit/s) to roughly 80–95% of your measured real-world rates (see [determining shaping rates](#determining-realistic-download-upload-shaping-rates) below).
4. Queue discipline: `cake` (recommended default) with script `piece_of_cake.qos`, or `fq_codel` with `simple.qos` if you need lower CPU overhead.
5. Under **Link Layer Adaptation**, set the correct per-packet overhead for your connection type (see [PPPoE/overhead](#uploaddownload-overhead-and-pppoe-considerations) below).
6. Save & Apply.

### CLI/UCI configuration

```sh
apk add sqm-scripts

uci set sqm.@queue[0].enabled='1'
uci set sqm.@queue[0].interface='<WAN_INTERFACE>'
uci set sqm.@queue[0].download='85000'      # kbit/s, example: 85 Mbit down
uci set sqm.@queue[0].upload='20000'        # kbit/s, example: 20 Mbit up
uci set sqm.@queue[0].qdisc='cake'
uci set sqm.@queue[0].script='piece_of_cake.qos'
uci set sqm.@queue[0].linklayer='ethernet'  # or 'atm' for ADSL-style links
uci set sqm.@queue[0].overhead='22'         # see overhead section below
uci commit sqm
/etc/init.d/sqm restart
```

> If `sqm.@queue[0]` doesn't exist yet (fresh install), use `uci add sqm queue` first, then set options on that new section, or edit via LuCI once and let it generate the section, then tune from CLI afterward.

### How to identify the correct WAN interface

Do not assume `eth0`/`wan` — confirm on your actual router:

```sh
ip -brief link show
uci show network | grep -E "\.device|\.ifname"
cat /etc/board.json | grep -A3 '"wan"'
```

Look specifically for the device backing the `wan` (and `wan6`, if you have a separate IPv6 WAN logical interface) UCI network interface — that underlying device name is what SQM needs, not necessarily the UCI logical name.

### Determining realistic download/upload shaping rates

1. Run a wired speed test **without SQM enabled** at a quiet time (e.g., a reputable speed-test CLI, or your ISP-provided expected sync rate for DSL/fiber).
2. Take **~85–95%** of the measured (or ISP-quoted sync) rate as your SQM download/upload figures. Cable/fiber connections with less overhead can often push toward 95%; DSL/PPPoE connections with more overhead and more variable sync should start closer to 80–85%.
3. Re-test with SQM active and a bufferbloat test (see below); nudge the numbers down further if latency under load is still poor, or up slightly if throughput loss feels excessive and latency stays good.

### Why shaping too close to max line rate reduces effectiveness

If SQM's configured rate is too close to (or above) the actual physical bottleneck, packets still queue up **in the upstream device** (ISP modem/ONT buffer) before reaching your shaper, so CAKE/fq_codel never gets the chance to manage that queue — bufferbloat returns. The whole mechanism depends on your shaper being the *first* and *tightest* bottleneck the traffic hits.

### Recommended starting values

- Start at 85% of measured throughput for both directions.
- For DOCSIS/cable and most fiber (no ATM framing), start with `linklayer=ethernet` and overhead `22` (a common conservative default) or `18` if you know there's no VLAN tagging involved; refine per your ISP's actual framing if known.
- For PPPoE, add ~8 bytes on top of your normal Ethernet overhead (commonly totalling ~26–34 bytes depending on VLAN tagging) — see below.
- For any ADSL/VDSL-over-ATM connection, use `linklayer=atm` and an overhead around 44 bytes.

> These are **starting points**, not universal truths — actual overhead depends on your specific ISP's framing. If uncertain, say so to yourself explicitly and treat the numbers as a tuning baseline, not gospel.

### How to test bufferbloat before and after SQM

- Use a bufferbloat-focused speed test (e.g., the Waveform Bufferbloat test, or the dslreports/Cloudflare-style speed tests that report latency-under-load / grades) from a wired client, run once **before** enabling SQM and once **after**, both while doing a simultaneous large upload+download.
- Compare the "unloaded" vs "loaded" latency numbers — a well-tuned SQM setup should keep loaded latency within roughly tens of milliseconds of idle latency, versus potentially hundreds to a thousand+ ms unshaped.

### How to tune SQM properly

1. Set download/upload to your baseline (85% of measured).
2. Run a loaded-latency test.
3. If bufferbloat is still bad → lower the rate a bit further (try 75–80%) and confirm your `linklayer`/`overhead` settings actually match your connection type.
4. If latency is great but throughput loss feels too aggressive → raise the rate in small increments (2–5%) and re-test until bufferbloat starts creeping back, then back off one increment.
5. Re-validate periodically — ISP sync rates and peering can drift over time (e.g., cable connections in particular can have rate changes).

### CAKE options and when to use them

Some commonly used CAKE parameters (set via the `script`/`qdisc_advanced` options in `/etc/config/sqm` or the LuCI advanced tab):

| Option | Purpose |
|---|---|
| `diffserv4` / `diffserv3` / `besteffort` | Selects CAKE's built-in priority-tin scheme based on DSCP markings; `besteffort` disables tin-based prioritization |
| `nat` | Tells CAKE to account for NAT when doing per-host fairness (usually needed on a router doing NAT) |
| `dual-srchost` / `dual-dsthost` | Per-host fairness based on source or destination address, useful on the LAN-facing/download side |
| `ack-filter` | Thins redundant TCP ACKs on asymmetric links to free upload capacity — useful on connections with much less upload than download |
| `overhead <n>` / `linklayer` | Per-packet framing compensation, see overhead discussion above |

Only enable options you understand the effect of — CAKE's defaults (`piece_of_cake.qos`, `diffserv4`, `nat`) are a reasonable starting point for most home routers.

### Upload/download overhead considerations & PPPoE

PPPoE adds encapsulation overhead on top of the base Ethernet frame. If your WAN connection is PPPoE:

- Add PPPoE's overhead into your `overhead` value (on top of any VLAN-tagging overhead your ISP uses).
- SQM should generally be applied to the PPPoE logical interface's underlying device where possible, so it shapes the actual encapsulated traffic — verify which device SQM is bound to matches your PPPoE session, not the raw Ethernet WAN port underneath it, if there's a difference in your setup.

### Interaction with hardware/software flow offloading

> ⚠️ **Important: Hardware flow offloading should generally be disabled when using SQM.**
>
> Flow offloading (hardware or software) exists specifically to **bypass** parts of the normal Linux networking stack — including qdiscs — to maximize raw NAT throughput. SQM depends on *every* packet passing through its qdisc so CAKE/fq_codel can shape and queue it. If flow offloading is active, offloaded flows skip that qdisc, meaning SQM either does nothing for those flows or behaves inconsistently. This is a well-documented conflict, not a rare edge case — LuCI itself flags flow offloading as experimental and "not fully compatible with QoS/SQM."
>
> **Practical rule:** if SQM is enabled on an interface, turn **off** both *Software Flow Offloading* and *Hardware Flow Offloading* under **Network → Firewall → General Settings** for that path. You are trading some raw maximum NAT throughput for the low-latency behavior SQM provides — that trade is the entire point of enabling SQM in the first place.

```sh
uci set firewall.@defaults[0].flow_offloading='0'
uci set firewall.@defaults[0].flow_offloading_hw='0'
uci commit firewall
/etc/init.d/firewall restart
```

### How to troubleshoot SQM when throughput becomes too low

1. Confirm flow offloading is off (see above) — leaving it on doesn't break SQM outright but the combination is unsupported and can behave unpredictably.
2. Re-check your configured download/upload numbers aren't set too conservatively.
3. Check CPU usage while under load (see below) — if a CPU core is pegged at 100% during the shaped traffic test, the CPU (not your configured rate) is your bottleneck.
4. Try `fq_codel` instead of `cake` temporarily to see if the overhead of CAKE's extra bookkeeping is the limiting factor.
5. Confirm `linklayer`/`overhead` match your actual connection type — wrong overhead assumptions distort effective shaped throughput.

### How to determine whether the router CPU is the bottleneck

```sh
# Watch live CPU usage while running a loaded throughput test from a LAN client
top -d 1

# Or, for a snapshot broken out by core
cat /proc/loadavg
mpstat -P ALL 1 5   # if 'sysstat' package is installed
```

If CPU usage (particularly for `ksoftirqd`, `cake_*` kernel threads, or the shaping-related kernel workers) sits near 100% on one or both cores while throughput is below your configured shaping rate, the CPU — not your SQM settings — is the limiting factor. At that point, either lower expectations for maximum shaped throughput, switch to `fq_codel`, or accept that SQM at full WAN-line-rate simply isn't realistic on this hardware for very high-speed connections.

---

## 3. QoS

### What QoS means

"QoS" (Quality of Service) here refers to **traditional traffic classification and prioritization** — explicitly marking or classifying certain traffic (e.g., VoIP, DNS, SSH) and giving it preferential treatment (priority scheduling, guaranteed minimum bandwidth, or dedicated queues), independent of any bufferbloat-focused shaping.

### Difference between QoS and SQM

| | SQM | Traditional QoS |
|---|---|---|
| Primary goal | Control bufferbloat / latency under load | Prioritize specific traffic classes over others |
| Mechanism | Rate shaping + AQM (CAKE/fq_codel) | Classification (DSCP/marks) + priority queuing/policing |
| Typical scope | Whole WAN link, automatically fair between flows | Specific rules per traffic type/host/port |
| CPU cost | Moderate | Can be low (simple marking) to significant (deep classification) |

### Whether users should use SQM or traditional QoS in common home-router scenarios

For the vast majority of home users, **SQM alone (with CAKE) solves the practical problem** — it keeps latency low under load and gives DSCP-aware traffic (many VoIP/video-call apps already mark their own packets) reasonable priority through CAKE's built-in tins, without any manual rule authoring. Traditional, hand-built QoS classification is mostly worth the added complexity when:

- You have a specific traffic type that needs guaranteed treatment beyond what CAKE's default tins give it (e.g., a security camera upload stream that must never be starved).
- You want per-device/per-port bandwidth guarantees rather than just fair-queuing behavior.

**Recommendation for the HX21:** start with SQM+CAKE only. Add manual QoS/DSCP marking rules only for specific, identified problems — not preemptively.

### Current OpenWrt packages and LuCI support

There isn't a single dedicated "QoS" apk package the way there is for `sqm-scripts` — traditional QoS on modern OpenWrt is generally implemented through:

- **DSCP marking/classification** in the firewall (`iptables`-style `mark`/`dscp` rules are now expressed as nftables rules via `/etc/config/firewall` traffic rules or raw `/etc/nftables.d/*.nft` snippets).
- **CAKE's own tin system** (`diffserv4`/`diffserv3`) as configured under SQM, which is often "enough" QoS for home use without a separate package.
- Community packages such as `qosify` for DSCP-based classification exist in some feeds — verify current availability with `apk search qosify` on your router, since availability varies by release/target and this is more niche than SQM.

```sh
apk update
apk search qosify     # confirm current availability/name on your target before relying on it
```

### Installation using apk

If you decide traditional classification is warranted:

```sh
apk update
apk add qosify           # if available for your target — verify with apk search first
apk add luci-app-qosify  # optional LuCI page, if published for your release
```

> Because package availability for niche QoS tools shifts between releases, treat this as "check before you plan around it," not a guaranteed component of every 25.12 image.

### Configuration options / traffic classification / DSCP and packet marking

A minimal DSCP-marking approach using the firewall's traffic-rule mechanism (mark outbound UDP for a VoIP/game server range as expedited-forwarding, for example) can be done via UCI:

```sh
uci add firewall rule
uci set firewall.@rule[-1].name='Mark-VoIP-EF'
uci set firewall.@rule[-1].src='lan'
uci set firewall.@rule[-1].proto='udp'
uci set firewall.@rule[-1].dest_port='<VOIP_PORT_RANGE>'
uci set firewall.@rule[-1].target='DSCP'   # exact target syntax depends on fw4 version — verify in LuCI's "Custom Rules"/"Traffic Rules" UI on your build
uci commit firewall
/etc/init.d/firewall restart
```

> The exact UCI option for DSCP remarking has evolved across firewall3→firewall4; the safest approach is to build the rule once in **LuCI → Network → Firewall → Traffic Rules** (which exposes a "DSCP" action where supported) and then read back the resulting `/etc/config/firewall` section with `uci show firewall` to see the exact syntax your build generated, rather than guessing.

### Practical examples

- **Gaming/VoIP:** rely on CAKE's `diffserv4` tin classification (already picks up common EF/CS5 markings many voice/game clients set) rather than hand-rolled rules, unless you find a specific app that doesn't mark its own traffic.
- **DNS:** typically low-volume and latency-sensitive already; CAKE's fair-queuing alone usually keeps DNS snappy even under load — dedicated DNS prioritization is rarely necessary.
- **SSH:** interactive SSH is small-packet and low-volume; again, usually fine under CAKE without special-casing.

### CPU/performance implications

Every additional classification rule is extra work per packet in the firewall/nftables path. On the HX21's modest CPU, keep manual QoS rule sets small and targeted; a long list of granular DSCP rules is a real (if usually small) CPU cost, and the marginal benefit over SQM+CAKE's automatic behavior is often not worth it for a home network.

### Should QoS and SQM be used together?

Yes, they're complementary rather than redundant: SQM/CAKE handles bufferbloat and general fairness; a *small, targeted* set of DSCP marking rules (or none at all, relying on CAKE's tins) handles any specific prioritization need CAKE's defaults don't cover. Avoid running a second, independent traffic-shaping layer (e.g., a separate `tc`-based QoS script) *underneath or alongside* SQM on the same interface — that's redundant complexity and can actively conflict with SQM's own qdisc.

---

## 4. WireGuard

### What WireGuard is

WireGuard is a modern VPN protocol built into the Linux kernel, designed to be simple, fast, and cryptographically opinionated (it doesn't offer a menu of ciphers — one modern, secure suite is used for everyone). On OpenWrt, WireGuard is integrated as a native network protocol (`proto wireguard`), configured through the same UCI `network` system as any other interface.

### Core concepts

- **Public/private keys:** Each WireGuard endpoint (interface or peer) has a Curve25519 keypair. The private key never leaves the device that generated it; only the public key is shared with peers. Authentication and key exchange both derive from this keypair — there's no separate username/password or certificate authority to manage.
- **Peers:** Each remote endpoint your WireGuard interface talks to is configured as a "peer" — the peer's public key, optionally its network endpoint (IP:port), and the set of IP ranges routed to/from it.
- **AllowedIPs:** Per-peer, this defines two things at once: (1) which source IPs are accepted as valid traffic *from* that peer, and (2) which destination IPs get routed *to* that peer. `0.0.0.0/0` (and `::/0` for IPv6) means "route everything" (full tunnel); a narrower range means "route only this subnet" (split tunnel).
- **Endpoint:** The peer's real-world `IP:port` to connect to. A peer that will always initiate (e.g., a phone behind NAT) can omit an endpoint on the server side; a peer you're dialing out to (e.g., a VPN provider or your home router from the road) needs its endpoint specified.
- **PersistentKeepalive:** WireGuard is normally silent when idle (no traffic = no packets), which is fine for a public server but breaks NAT/firewall state timeouts for a client behind NAT. Setting `persistent_keepalive` (commonly `25` seconds) makes that peer send an empty keepalive packet periodically to keep NAT/firewall mappings alive.
- **Handshakes:** WireGuard performs a fast, stealthy handshake (based on Noise protocol) roughly every 2 minutes while traffic flows, re-deriving session keys. "Latest handshake" age is the single most useful liveness indicator for a tunnel.

### Routing, MTU, IPv4/IPv6, DNS

- **Routing:** Traffic matching a peer's `AllowedIPs` is routed into the tunnel via policy routing rules WireGuard sets up automatically as part of the interface. On OpenWrt, this integrates with the standard `network` interface/route system.
- **MTU:** WireGuard adds ~60 bytes of overhead (UDP + WireGuard header) on top of the underlying path's MTU. A common safe default is **1420** for a typical 1500-byte-MTU IPv4 path; go lower (commonly 1400 or less) if your WAN itself has reduced MTU (e.g., PPPoE, some mobile/satellite links) or you see fragmentation/blackhole symptoms.
- **IPv4 and IPv6:** WireGuard is dual-stack capable — a single tunnel can carry both IPv4 and IPv6 payload regardless of whether the underlying transport (the UDP packets themselves) is IPv4 or IPv6. You assign both an IPv4 and IPv6 address to the interface if you want to route both families through it.
- **DNS considerations:** A WireGuard tunnel does *not* automatically change what DNS server your traffic uses — if you want to avoid DNS leaks (DNS queries going out your normal WAN instead of through the tunnel), you must explicitly configure the tunnel's DNS servers on the client and/or enforce it via firewall rules that block port 53 from leaving any interface except the tunnel. See the [Full-Tunnel VPN](#full-tunnel-vpn) section for a concrete kill-switch approach.

### Firewall zones and NAT/masquerading

WireGuard interfaces need to be placed into a firewall zone like any other interface — OpenWrt does not do this automatically. Typically:

- A **client** setup (router connecting *out* to a VPN) needs its `wg` interface zoned so LAN traffic is allowed to forward into it, and (for full tunnel) the `wan` zone should generally *not* also carry that traffic once you're relying on the tunnel.
- A **server** setup (router accepting *incoming* WireGuard connections) needs: the `wg` zone allowed to forward to `lan` (so remote clients can reach LAN devices), NAT/masquerading enabled on whichever zone remote clients' traffic needs to appear to originate from, and a port-forward/input rule on `wan` allowing the WireGuard UDP port in.

### Split tunnel vs. full tunnel

- **Split tunnel:** Only specific subnets (e.g., your home LAN, or a specific VPN provider's internal range) are routed through the tunnel via narrow `AllowedIPs`; everything else uses the normal WAN path. Lower overhead, but no "everything is protected" guarantee.
- **Full tunnel:** `AllowedIPs = 0.0.0.0/0, ::/0` (as appropriate) routes *all* traffic through the tunnel. Needed for "route my whole LAN through a VPN provider" setups; requires careful DNS and kill-switch handling (see below) to avoid leaks.

### Installation (apk)

```sh
apk update
apk add wireguard-tools        # wg / wg-quick CLI utilities
apk add luci-proto-wireguard   # LuCI: configure WireGuard interfaces under Network > Interfaces
apk add luci-app-wireguard     # LuCI: WireGuard status/overview page (optional but useful)
```

> The WireGuard kernel module itself ships as part of the OpenWrt kernel package set on modern targets including `mediatek/filogic`; `wireguard-tools` provides the userspace `wg` command used for key generation and status inspection. If `wg` reports the kernel module is missing, search `apk search wireguard` for a `kmod-wireguard`-style package matching your exact kernel build.

---

### WireGuard Client (router connects out to a remote server)

#### 1. Generate keys

```sh
mkdir -p /etc/wireguard
umask 077
wg genkey | tee /etc/wireguard/client-private.key | wg pubkey > /etc/wireguard/client-public.key
cat /etc/wireguard/client-public.key   # give this to the server operator
```

#### 2. Create the WireGuard interface

```sh
uci set network.wgclient=interface
uci set network.wgclient.proto='wireguard'
uci set network.wgclient.private_key="$(cat /etc/wireguard/client-private.key)"
uci add_list network.wgclient.addresses='<VPN_ADDRESS>/32'   # the address the VPN provider/server assigned you
uci commit network
```

#### 3. Add the remote peer (the server you're connecting to)

```sh
uci add network wireguard_wgclient
uci set network.@wireguard_wgclient[-1]=wireguard_wgclient
uci set network.@wireguard_wgclient[-1].public_key='<SERVER_PUBLIC_KEY>'
uci set network.@wireguard_wgclient[-1].endpoint_host='<VPN_ENDPOINT>'
uci set network.@wireguard_wgclient[-1].endpoint_port='51820'
uci set network.@wireguard_wgclient[-1].persistent_keepalive='25'
```

#### 4. Configure AllowedIPs — split tunnel example (route only a remote subnet)

```sh
uci add_list network.@wireguard_wgclient[-1].allowed_ips='10.10.10.0/24'
uci commit network
```

#### 5. Route ALL IPv4 traffic through WireGuard (full tunnel)

```sh
uci del network.@wireguard_wgclient[-1].allowed_ips  # clear any narrower entries first
uci add_list network.@wireguard_wgclient[-1].allowed_ips='0.0.0.0/0'
uci commit network
```

#### 6. Route IPv6 through WireGuard where appropriate

Only do this if your VPN endpoint actually supports IPv6 and you've been assigned an IPv6 address inside the tunnel; otherwise omit `::/0` and instead consider *blocking* outbound IPv6 on `wan` entirely to prevent IPv6 bypassing the tunnel (see [Full-Tunnel VPN](#full-tunnel-vpn)).

```sh
uci add_list network.wgclient.addresses='<VPN_IPV6_ADDRESS>/128'
uci add_list network.@wireguard_wgclient[-1].allowed_ips='::/0'
uci commit network
```

#### 7. Prevent DNS leaks (client side)

Point DNS at a resolver reachable *through* the tunnel, and don't let the router also hand out/advertise WAN-provided DNS to LAN clients:

```sh
uci set network.wgclient.peerdns='0'          # don't accept DNS from this proto automatically if not desired
uci -q delete dhcp.lan.dhcp_option
uci add_list dhcp.lan.dhcp_option='6,<VPN_DNS_SERVER>'
uci commit dhcp
/etc/init.d/dnsmasq restart
```

For a hard guarantee against leaks, add an explicit firewall rule blocking outbound port 53 from any interface except the tunnel — see [Full-Tunnel VPN](#full-tunnel-vpn) below.

#### 8. Configure firewall rules

```sh
uci add firewall zone
uci set firewall.@zone[-1].name='wgclient'
uci set firewall.@zone[-1].input='REJECT'
uci set firewall.@zone[-1].output='ACCEPT'
uci set firewall.@zone[-1].forward='REJECT'
uci set firewall.@zone[-1].masq='1'
uci set firewall.@zone[-1].network='wgclient'

uci add firewall forwarding
uci set firewall.@forwarding[-1].src='lan'
uci set firewall.@forwarding[-1].dest='wgclient'

uci commit firewall
/etc/init.d/firewall restart
```

#### 9. Bring the interface up and verify

```sh
ifup wgclient
# or, if it doesn't pick up new config:
ifdown wgclient && ifup wgclient
```

#### 10. Verify the tunnel

```sh
wg show                     # overall status: peer, endpoint, latest handshake, transfer
wg show wgclient latest-handshakes
ip addr show wgclient       # confirm the interface has the expected address
```

- **Check handshake status:** A recent (within the last couple of minutes, assuming keepalive is set) "latest handshake" timestamp from `wg show` means the tunnel is cryptographically alive.
- **Check RX/TX traffic:** `wg show wgclient transfer` shows bytes received/sent per peer — non-zero and growing confirms actual data flow, not just a handshake.
- **Check routes:** `ip route show table all | grep wgclient` and `ip rule show` to confirm policy routing is sending the intended traffic into the tunnel.

#### Troubleshoot connection failures

| Symptom | Check |
|---|---|
| No handshake ever | Correct `endpoint_host`/port? WAN firewall/NAT allowing outbound UDP? Correct public keys on both sides (not swapped)? |
| Handshake succeeds, no traffic passes | `AllowedIPs` mismatch between client and server, or firewall forwarding rule missing |
| Works then drops | NAT timeout — add/lower `persistent_keepalive` |
| Works locally, not after reboot | Interface not set to auto-start, or config not committed (`uci commit`) |

---

### WireGuard Server (router accepts remote connections)

#### 1. Generate server keys

```sh
mkdir -p /etc/wireguard
umask 077
wg genkey | tee /etc/wireguard/server-private.key | wg pubkey > /etc/wireguard/server-public.key
```

#### 2. Create the server interface

```sh
uci set network.wgserver=interface
uci set network.wgserver.proto='wireguard'
uci set network.wgserver.private_key="$(cat /etc/wireguard/server-private.key)"
uci set network.wgserver.listen_port='51820'
uci add_list network.wgserver.addresses='10.10.10.1/24'   # tunnel-internal subnet you're creating
uci commit network
```

#### 3. Generate a client keypair and (optionally) a pre-shared key

```sh
wg genkey | tee /etc/wireguard/client1-private.key | wg pubkey > /etc/wireguard/client1-public.key
wg genpsk > /etc/wireguard/client1-preshared.key   # optional extra symmetric layer
```

#### 4. Add the client as a peer, assign it an address

```sh
uci add network wireguard_wgserver
uci set network.@wireguard_wgserver[-1]=wireguard_wgserver
uci set network.@wireguard_wgserver[-1].description='client1-phone'
uci set network.@wireguard_wgserver[-1].public_key="$(cat /etc/wireguard/client1-public.key)"
uci set network.@wireguard_wgserver[-1].preshared_key="$(cat /etc/wireguard/client1-preshared.key)"
uci add_list network.@wireguard_wgserver[-1].allowed_ips='10.10.10.2/32'
uci set network.@wireguard_wgserver[-1].persistent_keepalive='25'
uci commit network
```

Repeat step 4 (new `uci add network wireguard_wgserver`, `[-1]` indexing) for each additional client, giving each a unique `/32` inside `10.10.10.0/24`.

#### 5. Configure the firewall zone

```sh
uci add firewall zone
uci set firewall.@zone[-1].name='wgserver'
uci set firewall.@zone[-1].input='ACCEPT'
uci set firewall.@zone[-1].output='ACCEPT'
uci set firewall.@zone[-1].forward='REJECT'
uci set firewall.@zone[-1].masq='1'
uci set firewall.@zone[-1].network='wgserver'
uci commit firewall
```

#### 6. Enable forwarding (LAN access) and NAT

```sh
# Allow remote WireGuard clients to reach your LAN
uci add firewall forwarding
uci set firewall.@forwarding[-1].src='wgserver'
uci set firewall.@forwarding[-1].dest='lan'

# Allow LAN devices to initiate back to WireGuard clients (needed for many app protocols)
uci add firewall forwarding
uci set firewall.@forwarding[-1].src='lan'
uci set firewall.@forwarding[-1].dest='wgserver'

uci commit firewall
```

#### 7. Open the WAN port for incoming WireGuard connections

```sh
uci add firewall rule
uci set firewall.@rule[-1].name='Allow-WireGuard-Inbound'
uci set firewall.@rule[-1].src='wan'
uci set firewall.@rule[-1].dest_port='51820'
uci set firewall.@rule[-1].proto='udp'
uci set firewall.@rule[-1].target='ACCEPT'
uci commit firewall
/etc/init.d/firewall restart
```

#### 8. Bring the interface up

```sh
ifup wgserver
wg show
```

#### 9. Connect a remote client

On the client device (phone, laptop, another router), configure:

```ini
[Interface]
PrivateKey = <CLIENT_PRIVATE_KEY>
Address = 10.10.10.2/32
DNS = 10.10.10.1

[Peer]
PublicKey = <SERVER_PUBLIC_KEY>
PresharedKey = <PRESHARED_KEY_IF_USED>
Endpoint = <YOUR_PUBLIC_IP_OR_DDNS>:51820
AllowedIPs = 10.10.10.0/24, <LAN_SUBNET>   # LAN access; use 0.0.0.0/0 for full tunnel through this router
PersistentKeepalive = 25
```

### Remote access example: reaching the HX21's LAN from outside

With the server config above, once a remote client connects and its `AllowedIPs` includes your `<LAN_SUBNET>` (e.g., `192.168.1.0/24`), and the router's forwarding rules (`wgserver → lan`) are in place, the remote client can reach LAN devices directly by their LAN IP — e.g., `ssh user@192.168.1.50` from the road, as if it were on-site. Ensure:

- The remote client's own `AllowedIPs` explicitly lists the LAN subnet (not just the WireGuard subnet), or LAN routes won't be pulled into its routing table.
- LAN devices don't have their own firewalls blocking the WireGuard subnet.
- If you need the reverse too (LAN devices initiating connections *to* remote clients), the `lan → wgserver` forwarding rule from step 6 covers that.

### Handling IPv4 and IPv6 on the server

If you want IPv6-capable clients:

```sh
uci add_list network.wgserver.addresses='fd00:10:10::1/64'   # ULA or your own IPv6 delegation
uci commit network
```

Then include an IPv6 range in each peer's `allowed_ips` and in the client's tunnel config as needed. Remember: NAT/masquerading concepts differ for IPv6 (usually routed, not NATed) — decide whether you actually want IPv6 clients getting globally routable addresses or staying on a NAT64/ULA-only internal scheme, based on your ISP's IPv6 delegation.

---

### Full-Tunnel VPN (route all LAN clients through a WireGuard provider/server)

This is the "route my whole home network through a VPN" pattern — different from the client/server sections above in that *LAN clients*, not just the router itself, are the ones whose traffic needs to go through the tunnel.

#### Firewall architecture

- Create the WireGuard client interface (see [WireGuard Client](#wireguard-client-router-connects-out-to-a-remote-server) above) with `AllowedIPs = 0.0.0.0/0` (and `::/0` if using IPv6 through the tunnel).
- Put that interface in its own zone (e.g., `vpn`), with `masq=1` so LAN traffic is NATed behind the tunnel's assigned address.
- Change the **LAN zone's forwarding** so LAN forwards to the `vpn` zone **instead of** (or in addition to, with careful rule ordering — see kill-switch below) the `wan` zone.

```sh
uci -q delete firewall.@forwarding[-1]   # remove any existing lan -> wan forwarding, carefully identify the right index first with `uci show firewall`
uci add firewall forwarding
uci set firewall.@forwarding[-1].src='lan'
uci set firewall.@forwarding[-1].dest='vpn'
uci commit firewall
```

#### Routing

WireGuard's own route/AllowedIPs handling takes care of directing `0.0.0.0/0` traffic into the tunnel once it's the only (or higher-priority) forwarding path from `lan`. Confirm with `ip route` and `ip rule` after applying.

#### DNS

Point LAN DHCP at a DNS server reachable *through* the tunnel (either the VPN provider's DNS, or your own resolver on the far end), and stop advertising the ISP-provided WAN DNS to LAN clients — otherwise DNS queries can leak outside the tunnel even while other traffic is correctly routed.

```sh
uci -q delete dhcp.lan.dhcp_option
uci add_list dhcp.lan.dhcp_option='6,<VPN_DNS_SERVER>'
uci commit dhcp
/etc/init.d/dnsmasq restart
```

#### Kill-switch considerations: what happens if the VPN goes down, and preventing accidental WAN fallback

By default, if you *also* leave a `lan → wan` forwarding rule in place (even a lower-priority one), OpenWrt's firewall will often still allow traffic out via `wan` if the `vpn` interface goes down and its route disappears — this is exactly the "leak on VPN failure" scenario a kill-switch prevents. To build a real kill-switch:

1. **Do not** leave a `lan → wan` forwarding rule active while relying on the VPN for full-tunnel behavior. Only `lan → vpn` should exist.
2. Explicitly **reject** `lan → wan` forwarding with a dedicated rule (rather than just omitting a permit rule), so there's no ambiguity if firewall reload ordering ever changes:

```sh
uci add firewall forwarding
uci set firewall.@forwarding[-1].src='lan'
uci set firewall.@forwarding[-1].dest='wan'
uci set firewall.@forwarding[-1].family='any'
# firewall4's `forwarding` sections don't take a target directly the way rules do;
# to hard-block, add an explicit reject *rule* instead of relying on forwarding-absence:
uci add firewall rule
uci set firewall.@rule[-1].name='Block-LAN-WAN-Fallback'
uci set firewall.@rule[-1].src='lan'
uci set firewall.@rule[-1].dest='wan'
uci set firewall.@rule[-1].target='REJECT'
uci commit firewall
/etc/init.d/firewall restart
```

3. If you also routed IPv6 through the tunnel, make sure **IPv6 has an equivalent hard block on `wan`** — a very common leak is IPv4 correctly tunneled while IPv6 quietly continues going out the normal WAN path because it was never addressed. If you are *not* routing IPv6 through the VPN at all, simply disabling/not-forwarding IPv6 out of `lan` entirely may be the pragmatic choice, rather than leaving it to go direct to `wan`.

#### How to test for IP/DNS leaks

- From a LAN client, visit an "IP/DNS leak test" style page and confirm the reported public IP matches your VPN endpoint, not your real WAN IP.
- Deliberately bring the `vpn`/WireGuard interface down (`ifdown wgclient`) on the router and confirm LAN clients **lose internet entirely** rather than silently falling back to direct WAN — that's your kill-switch working as intended.
- Check DNS specifically (a general "leak test" page's IP result doesn't guarantee DNS is also going through the tunnel) — many dedicated DNS-leak-test sites report which resolvers actually answered your query.

---

## 5. Firewall (firewall4 / nftables)

### Current architecture

Since OpenWrt 22.03, the default firewall backend is **firewall4 (`fw4`)**, which targets the kernel's **nftables** subsystem instead of the older iptables-based `firewall3`/`fw3`. The **UCI configuration format** (`/etc/config/firewall`) is unchanged between fw3 and fw4 — existing zone/rule/forwarding syntax works as-is; fw4 just compiles it into nftables rules instead of iptables rules under the hood. On OpenWrt 25.12, firewall4 is the only supported backend.

fw4 builds a single nftables table, `inet fw4`, containing chains for input/output/forward hooks, NAT, and (where enabled) hardware-offload flowtables — you generally don't hand-edit that table directly; you configure UCI and let `fw4` regenerate it.

### UCI firewall configuration: zones, forwarding, policies

- **Zones** group interfaces that share a trust level (e.g., `lan`, `wan`, and any VPN zones you add) and set default input/output/forward policies for that zone.
- **Forwarding** sections define explicit permission for traffic to cross from one zone to another (e.g., `lan → wan`). Without an explicit forwarding entry, cross-zone forwarding is denied by default.
- **Rules** are specific exceptions/permits/denies layered on top of zone policy (e.g., "allow inbound UDP 51820 on wan" for WireGuard).
- **Redirects** implement NAT/port-forwarding (DNAT), e.g., forwarding a WAN port to an internal LAN server.

Example baseline (illustrative — your actual `/etc/config/firewall` will already contain a working `lan`/`wan` setup from your device's default image; treat this as a reference, not something to blindly re-paste):

```uci
config defaults
	option synflood_protect '1'
	option input 'REJECT'
	option output 'ACCEPT'
	option forward 'REJECT'

config zone
	option name 'lan'
	list network 'lan'
	option input 'ACCEPT'
	option output 'ACCEPT'
	option forward 'ACCEPT'

config zone
	option name 'wan'
	list network 'wan'
	list network 'wan6'
	option input 'REJECT'
	option output 'ACCEPT'
	option forward 'REJECT'
	option masq '1'
	option mtu_fix '1'

config forwarding
	option src 'lan'
	option dest 'wan'
```

### NAT/masquerading and port forwarding

- `option masq '1'` on a zone enables masquerading (source NAT) for traffic leaving that zone — this is what lets your whole LAN share one public WAN IP.
- Port forwarding (DNAT) is a `config redirect` section, mapping an external WAN port to an internal LAN IP:port.

```uci
config redirect
	option name 'Forward-SSH-to-server'
	option src 'wan'
	option src_dport '2222'
	option dest 'lan'
	option dest_ip '192.168.1.50'
	option dest_port '22'
	option proto 'tcp'
```

### WireGuard firewall integration

Covered in detail in [section 4](#4-wireguard) — the short version: every WireGuard interface needs its own zone (or needs to be added into an existing zone's `network` list), an explicit forwarding rule to/from whichever zones it should exchange traffic with, and (for a server) an input rule opening its listen port on `wan`.

### Useful commands

```sh
# Reload firewall after a config change (fast, applies UCI changes)
fw4 reload
# or:
/etc/init.d/firewall reload

# Full restart (heavier-handed than reload)
/etc/init.d/firewall restart

# Print the complete generated nftables ruleset fw4 built from your UCI config
fw4 print

# Show a zone/rule summary
fw4 status

# Inspect the raw, currently-loaded nftables ruleset directly from the kernel
nft list ruleset

# Inspect just the fw4 table
nft list table inet fw4
```

### Testing connectivity / troubleshooting firewall issues

```sh
# Confirm the firewall service is actually running
/etc/init.d/firewall status  # (or check via 'ps' / 'ubus call' depending on init script support)

# Watch live log output while testing a connection (blocked packets, if logging is enabled)
logread -f

# From a LAN client, test whether a specific forwarded/blocked port behaves as expected
nc -zv <TARGET_IP> <PORT>
```

> ⚠️ **Warning:** Always test firewall changes over a **wired LAN connection**, not through the interface/zone you're modifying, and not solely over the VPN tunnel you just built. An incorrect zone/rule change can lock you out of both LuCI and SSH simultaneously. Keep a console/physical-access fallback plan (or OpenWrt's failsafe boot mode) available.

For advanced use cases the UCI abstraction doesn't cover, raw nftables snippets can be dropped into `/etc/nftables.d/*.nft`, and fw4 will load them automatically after generating the main ruleset.

---

## 6. Network Diagnostics

Each command below is explained by what it's actually useful for, not just listed.

| Command | What it tells you |
|---|---|
| `ip addr` | Current IPv4/IPv6 addresses on every interface — confirms an interface actually got an address (DHCP, static, or VPN-assigned) |
| `ip route` | The IPv4 routing table — confirms which interface/gateway traffic to a given destination will actually use; critical for diagnosing "VPN traffic isn't going through the tunnel" |
| `ip -6 route` | Same, for IPv6 — essential when checking for IPv6 VPN leaks (see [Full-Tunnel VPN](#full-tunnel-vpn)) |
| `ip rule` | Policy routing rules — shows *which* routing table applies to which traffic; WireGuard and multi-WAN setups often rely on this beyond the plain main table |
| `wg show` | WireGuard tunnel status: peers, endpoints, latest handshake age, transfer counters — the primary WireGuard health check |
| `logread` | The system log (persisted across most runtime events) — check here first for service startup failures, firewall issues, DHCP problems |
| `logread -f` | Follow the log live — useful while reproducing an issue in real time |
| `ubus` | OpenWrt's internal RPC/service bus — `ubus call network.interface.<name> status` gives structured JSON status for any interface, often more precise than `ip addr` alone |
| `uci show <config>` | Dump the effective UCI configuration for a subsystem (`network`, `firewall`, `sqm`, `dhcp`, etc.) — confirms what's actually configured vs. what you *think* you configured |
| `nft list ruleset` | The actual, currently-active nftables rules generated by fw4 — the ground truth for what the firewall is really doing |
| `ping` / `ping6` | Basic reachability + round-trip latency to a target — first step in isolating "is this a routing problem or something higher up" |
| `traceroute` / `traceroute6` | Hop-by-hop path to a destination — useful for spotting where traffic is being dropped or unexpectedly routed (e.g., confirming VPN egress vs. direct WAN egress) |
| `nslookup` | Basic DNS query tool available in OpenWrt's busybox — confirms whether a name resolves and via which resolver; useful for catching DNS leaks or resolver misconfiguration |

Relevant OpenWrt-specific service commands:

```sh
/etc/init.d/network reload     # Re-apply network config without a full restart
/etc/init.d/network restart    # Full network stack restart (more disruptive)
/etc/init.d/dnsmasq restart    # Restart DHCP/DNS service after dhcp config changes
/etc/init.d/sqm restart        # Re-apply SQM config
/etc/init.d/firewall reload    # Re-apply firewall config
ubus call network.interface dump   # Structured status for every configured interface at once
```

---

## 7. Performance / Benchmarking

### What to benchmark, and how

| What | How |
|---|---|
| Normal WAN throughput | Wired speed test from a LAN client, SQM/QoS disabled, flow offloading as you'd normally run it |
| SQM throughput | Same test with SQM enabled — compare against the unshaped baseline to see the throughput cost of shaping |
| WireGuard throughput | `iperf3` between a device behind the WireGuard tunnel and a server on the far end, comparing against non-tunneled throughput on the same path |
| CPU usage | `top -d 1` or `mpstat -P ALL 1 5` while a throughput test is running, to see whether a core is saturated |
| Latency under load | A bufferbloat-style test (loaded vs. idle ping/latency) — see [SQM](#2-sqm--smart-queue-management) |
| Bufferbloat | Same as above; look specifically at the "loaded" latency grade/number, not just raw throughput |
| LAN throughput | `iperf3` between two wired LAN clients through the router, to isolate switch/CPU-forwarding performance from WAN-related factors entirely |

### Identifying the actual bottleneck

Work through these in order — each step isolates one more variable:

1. **ISP connection:** Test with a device connected *directly* to the ISP's modem/ONT (bypassing the HX21 entirely), if feasible. If throughput is already capped there, nothing on the router will fix it.
2. **Ethernet:** Confirm link negotiation is actually Gigabit (`ethtool <iface>` if available, or check LuCI's interface status) — a cable/port negotiating at 100 Mbps will silently cap everything downstream.
3. **Wi-Fi:** Repeat throughput tests over a wired connection first; if wired is fine but Wi-Fi is poor, the issue is RF/Wi-Fi-specific (channel congestion, distance, client capability), not the router's routing/CPU path.
4. **CPU:** Watch `top`/`mpstat` during a maximum-throughput test. Sustained near-100% on the relevant core(s) while throughput plateaus below expected is a strong signal of CPU-bound forwarding (common once SQM/CAKE or WireGuard encryption enters the picture on this hardware class).
5. **SQM/CAKE:** Temporarily disable SQM and re-test; if throughput jumps back to near line-rate, SQM's shaping (by design) or CPU cost (if CPU-bound) is the limiting factor for that specific test — expected behavior, just confirm it matches your configured shaping rate.
6. **WireGuard encryption:** Compare throughput with and without the tunnel on the same path; WireGuard's ChaCha20-Poly1305 encryption is CPU work, and on a dual-core A53 this can become the ceiling well below your raw WAN speed at high rates.
7. **DNS:** Slow *page loads* despite fine raw throughput often trace to slow DNS resolution, not routing — test with `time nslookup <domain>` or a raw throughput tool that doesn't depend on DNS (an IP-address-based `iperf3` test) to rule this in/out.
8. **Firewall:** An unusually large or inefficient custom ruleset can add per-packet CPU cost — check `nft list ruleset` for length/complexity if CPU usage is high even without SQM/WireGuard active.
9. **Routing:** Confirm via `ip route`/`ip rule` that traffic is actually taking the path you expect — a misrouted test (e.g., accidentally going through a VPN you forgot was still up) will produce confusing, artificially low numbers.

---

## 8. Backup / Recovery

### Backing up OpenWrt configuration

LuCI: **System → Backup / Flash Firmware → Generate archive** produces a `.tar.gz` of `/etc/config/*` and other files listed in `/etc/sysupgrade.conf`.

CLI equivalent:

```sh
sysupgrade -b /tmp/backup-$(date +%Y%m%d).tar.gz
```

Copy that archive off the router immediately (`scp`, or download via LuCI) — a backup that only exists on the router's own flash doesn't protect you against a failed flash or bricked device.

### What configuration files are important

Everything under `/etc/config/` (the UCI config directory) is the core of your setup:

| File | Covers |
|---|---|
| `/etc/config/network` | Interfaces, WireGuard tunnels, static routes |
| `/etc/config/firewall` | Zones, forwarding, rules, port forwards |
| `/etc/config/dhcp` | DHCP, DNS (dnsmasq) settings |
| `/etc/config/wireless` | Wi-Fi radios/SSIDs |
| `/etc/config/sqm` | SQM shaping configuration |
| `/etc/config/system` | Hostname, timezone, misc system settings |

Also worth preserving outside the default backup scope: `/etc/wireguard/*` private keys (only if stored outside UCI's own `private_key` option, which is itself included in `/etc/config/network` and thus in a standard backup) and any custom scripts under `/root` or `/etc/uci-defaults`.

### How to restore configuration

LuCI: **System → Backup / Flash Firmware → Restore backup**, upload the `.tar.gz`, then reboot.

CLI:

```sh
sysupgrade -r /tmp/backup-20260101.tar.gz   # extracts into the running system
reboot
```

> Restoring a backup from a *different* OpenWrt version or a device with different interface names can produce a broken or partially-applied config — prefer restoring onto the same release/build family you backed up from, and review `uci show network`/`uci show firewall` afterward for anything obviously stale.

### How to safely experiment with networking/firewall/WireGuard

- Take a fresh backup immediately before any risky change.
- Use `uci changes` to review pending, uncommitted changes before running `uci commit`.
- Prefer `reload` over `restart` where available (less disruptive), and test one change at a time rather than batching several unverified changes.
- For firewall/network changes made remotely, consider scheduling an automatic revert as a safety net:

```sh
# Schedule a reboot in 5 minutes as a safety net; cancel it once you've confirmed the change works
( sleep 300 && reboot ) &
```

  (Cancel by finding and killing that background job, or simply reboot manually once satisfied — this is a crude but effective "if I get locked out, the router will save itself" pattern.)

### How to recover from a bad configuration

1. **If SSH/LuCI is still reachable:** revert the specific `/etc/config/*` file from your last known-good backup, or restore the full backup archive, then `reboot` or reload the relevant service.
2. **If locked out but the router still boots normally:** use OpenWrt's **failsafe boot mode** (typically: power on, watch for the flashing status LED pattern, press/hold the reset button during that specific window — exact timing is firmware/build-dependent, check your build's documentation) to get a minimal shell with networking, then fix or reset the offending config.
3. **If failsafe isn't reachable either:** fall back to a full reflash via the bootloader/recovery method documented for the HX21/LC-HX3001 port (see [Sources](#sources)) — this is why having a tested, working recovery path *before* you start experimenting matters.

### Safe upgrade practices / sysupgrade considerations

- **Prefer Attended Sysupgrade (ASU)** where available (LuCI, included by default in 25.12) — it rebuilds a firmware image that already contains your currently-installed extra packages, avoiding the "upgrade firmware, lose all my packages" problem.
- Always back up configuration immediately before a sysupgrade, even though `sysupgrade` normally preserves `/etc/config` across compatible upgrades — "normally" is not "always," especially across a major release boundary (e.g., 24.10 → 25.12, which also changes the package manager).
- Read the release notes' **known issues / regressions** section for your target release before upgrading — interface renames, changed defaults, or target-specific quirks are called out there (e.g., some devices had WAN interface names change between releases).
- Keep the **previous firmware image** downloaded locally if possible, so you can revert via the bootloader/recovery method if the new version misbehaves on your specific hardware revision.

---

## 9. Recommended Configuration Profiles

Each profile below assumes you start from a clean default LAN/WAN setup and layer the described features on top; see the relevant sections above for full command sequences.

### Basic Router

**What it does:** Stock LAN/WAN routing, default firewall, no SQM, no VPN — the out-of-the-box behavior most people never need to change for simple home use.

- **Advantages:** Zero added CPU load, maximum raw throughput, simplest to troubleshoot.
- **Disadvantages:** No bufferbloat protection — video calls/gaming can suffer under any concurrent heavy upload/download.
- **CPU impact:** Minimal.
- **Use when:** Your household doesn't do much simultaneous heavy-upload + latency-sensitive activity, or you're not chasing every last bit of responsiveness.
- **Don't use when:** You've noticed laggy calls/games whenever someone else is uploading/downloading — that's exactly what SQM fixes.

### Low-Latency Router

**What it does:** SQM with CAKE enabled on the WAN interface, hardware/software flow offloading disabled, shaping rates tuned to ~85% of measured throughput.

- **Advantages:** Dramatically reduced latency under load; noticeably smoother gaming/video calls even while others saturate the link.
- **Disadvantages:** Somewhat reduced maximum throughput (you're intentionally shaping under line rate); added CPU load.
- **CPU impact:** Moderate — budget for real headroom loss at higher WAN speeds (see [CPU bottleneck diagnosis](#how-to-determine-whether-the-router-cpu-is-the-bottleneck)).
- **Use when:** Latency-sensitive use (gaming, video calls, VoIP) matters to your household.
- **Don't use when:** Your WAN speed is high enough, and your CPU-headroom testing shows you can't shape it without a CPU bottleneck making things worse, not better — in that edge case, consider `fq_codel` instead of `cake`, or accept the trade-off consciously.

### WireGuard Client Router

**What it does:** Router-initiated WireGuard tunnel to a remote server/VPN provider, either full- or split-tunnel.

- **Advantages:** Every device on the LAN gets VPN protection without per-device client software (full tunnel), or selective access to a remote network (split tunnel).
- **Disadvantages:** Adds encryption CPU cost; full tunnel needs careful DNS/kill-switch handling to avoid leaks; single point of failure for all LAN internet access if full-tunnel and the VPN goes down without a kill-switch consideration (or the reverse — total outage if the kill-switch fires and the VPN is unreachable).
- **CPU impact:** Real but generally moderate at typical home-broadband speeds; can become the throughput ceiling at higher WAN speeds (see [Full-Tunnel VPN](#full-tunnel-vpn) and [benchmarking](#7-performance--benchmarking)).
- **Use when:** You want network-wide VPN coverage without configuring every client device individually.
- **Don't use when:** Only one or two devices need VPN access — per-device WireGuard clients avoid router CPU load and avoid making the whole LAN dependent on the tunnel's health.

### Remote Access Router

**What it does:** WireGuard server on the router, allowing you (or trusted others) to connect in from anywhere and reach the LAN as if physically present.

- **Advantages:** Secure remote access to home devices/services without exposing them individually to the internet.
- **Disadvantages:** Requires opening one WAN UDP port; requires a stable way to find your home IP (static IP or dynamic DNS) if your ISP doesn't provide a fixed address.
- **CPU impact:** Low at typical remote-access usage levels (occasional access, not sustained high-throughput transfer).
- **Use when:** You want to reach home devices/services (NAS, cameras, other LAN hosts) while away.
- **Don't use when:** You don't have a reliable way to reach the router's public IP (heavy CGNAT with no port-forwarding option from your ISP) — in that case a relay/reverse-tunnel approach outside the scope of this guide may be needed instead.

### VPN + SQM

**Should SQM operate before or after the VPN tunnel, and what are the practical limitations?**

SQM should shape at the **physical WAN interface**, i.e., "before/outside" the VPN tunnel from the perspective of the raw link, so it's managing the actual bottleneck (your real internet uplink), not an already-encapsulated, already-encrypted stream where CAKE can no longer usefully classify inner traffic (it's all opaque WireGuard UDP payload from the outside).

**Practical limitations:**

- WireGuard's own overhead (~60 bytes per packet) effectively reduces your usable payload throughput at a given shaped rate — when tuning SQM's overhead/linklayer settings for a WAN interface that's primarily carrying tunneled traffic, you may want to account for this extra encapsulation on top of your normal Ethernet/PPPoE overhead, though the practical effect is usually small relative to your overall shaping margin.
- You cannot classify/prioritize *inside* the tunnel from the WAN-facing SQM instance — all WireGuard traffic looks like uniform encrypted UDP from the outside. If you need CAKE-tin-style prioritization of specific inner traffic (e.g., prioritize a video call over a bulk transfer, both going through the tunnel), that has to happen *before* encryption — i.e., DSCP-marking or an SQM instance on the LAN-facing side/WireGuard interface itself — which is more advanced and CPU-costly setup, and not something to add without a specific, identified need.
- Running both WireGuard encryption and CAKE shaping simultaneously stacks two CPU-bound workloads on the same modest CPU — test throughput and latency under load specifically for this combination rather than assuming either feature's individual benchmark numbers will hold when combined.

---

## 10. Troubleshooting

For each problem: **Symptoms → Likely causes → Diagnostic commands → Fix → Verification.**

| # | Problem | Symptoms | Likely Causes | Diagnostic Commands | Fix | Verification |
|---|---|---|---|---|---|---|
| 1 | `apk` cannot find package | "unable to select packages" / "no such package" | Stale index; package not published for your target; typo | `apk update`; `apk search <term>` | Confirm exact name via search; re-run `apk update` before `apk add` | `apk info <pkg>` shows the package |
| 2 | Package architecture mismatch | Install fails with an arch/ABI error | Package built for a different CPU arch/target than `mediatek/filogic`/`aarch64` | `cat /etc/openwrt_release`; `apk search <pkg>` (only shows compatible results) | Only install packages found via `apk search` on the router itself, not copy-pasted from unrelated-target docs | Package installs cleanly |
| 3 | Kernel module mismatch | Feature (Wi-Fi, offload, WireGuard) fails to load after a partial upgrade | Kernel and kmod packages out of sync (e.g., after an unsupported partial `apk upgrade`) | `dmesg \| tail`; `logread \| grep -i module` | Perform a full sysupgrade to a consistent firmware build rather than patching individual kmod packages | `lsmod` shows the module loaded; feature works |
| 4 | SQM reduces speed dramatically | Throughput far below configured shaping rate | CPU-bound CAKE processing; overhead/linklayer misconfigured; flow offloading still enabled alongside SQM | `top -d 1` during test; check `uci show sqm` | Disable flow offloading; lower shaping rate to realistic CPU-bound ceiling; try `fq_codel` instead of `cake` | Re-test throughput at new settings |
| 5 | CAKE causes high CPU usage | One/both cores near 100% under load | Expected CPU cost of CAKE at your configured rate/traffic mix | `top -d 1`; `mpstat -P ALL 1 5` | Switch to `fq_codel`; lower shaped rate; consider CAKE_MQ if available on your build | CPU usage drops to sustainable levels |
| 6 | Bufferbloat is still bad | Loaded-latency test still shows large delay increase with SQM "on" | SQM not actually enabled/applied; shaped rate too close to real line rate; wrong interface targeted | `uci show sqm`; `tc qdisc show` on the WAN device; re-check with [bufferbloat test](#how-to-test-bufferbloat-before-and-after-sqm) | Confirm SQM is enabled on the *correct* device; lower shaped rate further | Re-run loaded-latency test |
| 7 | WireGuard handshake never occurs | `wg show` shows no "latest handshake" | Wrong endpoint/port; keys swapped or mistyped; WAN firewall blocking outbound/inbound UDP; NAT on either end blocking | `wg show`; `logread \| grep wireguard`; confirm keys with `wg pubkey < privatekey` | Re-verify endpoint, port, and both public keys; confirm firewall rule (server) or outbound path (client) | `wg show` shows a recent handshake timestamp |
| 8 | WireGuard connects but internet doesn't work | Handshake succeeds, but no traffic reaches destinations | Missing/incorrect forwarding rules; `AllowedIPs` too narrow; masquerading not enabled; MTU issue | `wg show <iface> transfer`; `uci show firewall`; `ping` through the tunnel to the peer's tunnel IP first | Add/fix forwarding rules; verify `masq=1` on the egress zone; lower MTU if large packets fail but small ones succeed | Ping succeeds to tunnel peer and beyond |
| 9 | LAN clients cannot access the VPN | Client traffic never enters tunnel | Missing `lan → vpn`/`wgclient` forwarding; routing table not pointing at tunnel | `ip route`; `uci show firewall \| grep forwarding` | Add forwarding rule; confirm route via `ip route get <target>` | `ip route get <target>` shows the tunnel interface |
| 10 | VPN clients cannot access LAN | Remote client connects but can't reach LAN devices | Missing `wgserver → lan` forwarding; remote client's `AllowedIPs` doesn't include LAN subnet | `uci show firewall`; check client-side WireGuard config | Add forwarding rule server-side; add LAN subnet to client's `AllowedIPs` | Remote client can ping/reach LAN devices |
| 11 | DNS leaks | Public DNS-leak test shows ISP resolver instead of VPN's | LAN DHCP still advertising WAN DNS; no firewall block on stray port 53 | `nslookup` from a client; check `uci show dhcp`; DNS-leak-test site | Point DHCP DNS option at tunnel-reachable resolver; add explicit reject rule for outbound 53 except via tunnel | Leak test shows only the intended resolver |
| 12 | IPv6 bypassing the VPN | IPv4 correctly tunneled, IPv6 leak-test shows real IP | IPv6 never routed into tunnel, or `wan` still allows direct IPv6 egress | `ip -6 route`; IPv6-specific leak-test site | Route `::/0` through tunnel if supported, or hard-block IPv6 egress on `lan → wan` | IPv6 leak test shows tunnel result or "no IPv6" (if blocked) |
| 13 | Internet stops when WireGuard is enabled | LAN loses internet as soon as tunnel interface comes up | Full-tunnel routing active but tunnel/server unreachable; kill-switch working as designed but VPN itself is down | `wg show` (handshake?); `ping <VPN_ENDPOINT>` from router | Fix underlying tunnel connectivity (see problem 7); this may be *correct* kill-switch behavior, not a bug | Tunnel handshakes again; internet restored through it |
| 14 | Firewall blocks WireGuard | Inbound connections never reach the server | Missing WAN input rule for the listen port; ISP/upstream also blocking the UDP port | `uci show firewall \| grep -A5 wireguard`; test port reachability from an external network | Add the WAN input rule for the correct UDP port; confirm ISP doesn't block/NAT that port (CGNAT) | External connection attempt reaches and handshakes |
| 15 | Router becomes unreachable after a firewall/network change | SSH/LuCI stop responding after `uci commit`/reload | Bad zone/policy change, especially on `lan`/`input` | Physical/console access; failsafe boot | Restore from backup; fix the offending section; re-apply carefully, testing incrementally | SSH/LuCI reachable again |
| 16 | High CPU usage (general) | Router sluggish, dropped packets, high load average | Combination of SQM + WireGuard + logging + extensive firewall rules simultaneously; possibly a runaway process | `top -d 1`; `cat /proc/loadavg`; `logread \| tail` | Identify and reduce the specific CPU-heavy feature combination (see [benchmarking](#7-performance--benchmarking)) | Load average and `top` return to normal |
| 17 | Wi-Fi performance issues | Slow/unstable Wi-Fi despite fine wired performance | Channel congestion; client capability mismatch; distance/obstruction; not a routing/CPU issue at all | Wired-vs-wireless throughput comparison (see [benchmarking](#7-performance--benchmarking)); Wi-Fi scan for channel congestion | Change channel/width; reposition; update client drivers | Wireless throughput test improves |
| 18 | MTU problems | Some sites/services fail to load (especially over VPN or PPPoE) while basic pings work | Path MTU issues combined with blocked ICMP fragmentation-needed messages; WireGuard overhead not accounted for | Test with progressively smaller `ping -s <size> -M do <target>` | Lower interface/tunnel MTU (commonly 1420 or lower for WireGuard, adjusted further for PPPoE) | Previously failing connections succeed |
| 19 | Routing conflicts | Traffic goes to the wrong interface/tunnel | Overlapping `AllowedIPs`/routes between multiple VPNs or static routes; policy routing rule ordering | `ip route`; `ip rule`; `ip route get <target>` | Adjust `AllowedIPs`/route specificity, or explicit route priorities | `ip route get <target>` shows expected interface |
| 20 | DNS problems (general) | Name resolution fails or is slow, independent of VPN | dnsmasq misconfigured/crashed; upstream resolver unreachable; leftover stale DHCP DNS options | `logread \| grep dnsmasq`; `nslookup <domain> <resolver>` | Restart dnsmasq; fix `/etc/config/dhcp` DNS options; test resolver directly | `nslookup` resolves correctly and quickly |

---

## 11. Security

- **SSH security:** Disable password authentication in favor of key-based auth once you've confirmed key login works (`/etc/config/dropbear`, `option PasswordAuth '0'` after adding your public key to `/etc/dropbear/authorized_keys`). Consider a non-default listening port only as a minor obscurity measure, not a substitute for key auth.
- **Disable unnecessary services:** Audit `/etc/init.d/` and disable anything you don't actually use (`/etc/init.d/<service> disable`) — every running service is attack surface and RAM/CPU usage.
- **Firewall defaults:** Leave `input=REJECT`/`forward=REJECT` on the `wan` zone as shipped; only add specific, narrow `ACCEPT` rules for services you deliberately expose (e.g., the WireGuard server port).
- **WireGuard key protection:** Private keys should be readable only by root (`umask 077` before generating, as shown throughout this guide); never transmit a private key over an insecure channel; treat a leaked private key as equivalent to a leaked password — regenerate and redistribute the corresponding peer config.
- **Avoid exposing LuCI to WAN.** There should be no `wan`-zone input rule permitting access to LuCI's port; manage the router only from the LAN or over the WireGuard tunnel.
- **Avoid exposing SSH to WAN** directly; if you need remote CLI access, prefer reaching it *through* the WireGuard tunnel rather than opening SSH on `wan`.
- **Strong passwords:** Set a strong root password (`passwd`) even if you also use key-based SSH auth, since it also gates LuCI web login.
- **Backup security:** Configuration backups contain private keys (WireGuard, potentially Wi-Fi PSKs) — store backups somewhere access-controlled, not in a public repository or unencrypted cloud folder.
- **Principle of least privilege:** Scope firewall rules, forwarding, and WireGuard `AllowedIPs` as narrowly as actually needed — a full-LAN-open WireGuard peer or a wide-open port forward is more exposure than most use cases require.
- **Safe package installation / avoiding random third-party feeds:** Stick to the official OpenWrt package feeds configured by default in `/etc/apk/repositories`. Adding third-party feeds means trusting that feed's maintainer with root-equivalent code execution on your router — only do so for feeds you specifically trust and understand, and prefer well-known, actively maintained community feeds over obscure ones.
- **Safe handling of private keys:** Never paste private keys into chat tools, tickets, or public forum posts when asking for help — redact them and share only public keys, which is all anyone else legitimately needs to configure a peer relationship with you.

---

## Command Cheat Sheet

### apk / package management

```sh
apk update                      # Refresh package index
apk add <pkg>                   # Install a package
apk del <pkg>                   # Remove a package
apk search <term>                # Search available packages
apk list                         # List all available packages
apk list -I                      # List installed packages
apk list -u                      # List upgradable packages
apk info <pkg>                   # Package info
apk info -L <pkg>                # Files owned by a package
apk -U add <pkg>                 # Force refresh index, then install
```

### SQM

```sh
apk add sqm-scripts luci-app-sqm
uci set sqm.@queue[0].enabled='1'
uci set sqm.@queue[0].interface='<WAN_INTERFACE>'
uci set sqm.@queue[0].download='<KBIT>'
uci set sqm.@queue[0].upload='<KBIT>'
uci set sqm.@queue[0].qdisc='cake'
uci commit sqm && /etc/init.d/sqm restart
tc qdisc show                    # Inspect active qdiscs
```

### WireGuard

```sh
apk add wireguard-tools luci-proto-wireguard luci-app-wireguard
umask 077 && wg genkey | tee privkey | wg pubkey > pubkey
wg genpsk > presharedkey
wg show                          # Status: peers, handshakes, transfer
wg show <iface> latest-handshakes
wg show <iface> transfer
ifup <iface> / ifdown <iface>    # Bring interface up/down
```

### Firewall

```sh
fw4 reload                       # Apply UCI firewall changes
fw4 print                        # Show generated nftables ruleset
fw4 status                       # Zone/rule summary
nft list ruleset                 # Raw active nftables rules
/etc/init.d/firewall restart
```

### Networking / diagnostics

```sh
ip addr
ip route
ip -6 route
ip rule
ping / ping6 <target>
traceroute / traceroute6 <target>
nslookup <domain>
logread [-f]
ubus call network.interface dump
uci show <config>
```

### Backup / recovery

```sh
sysupgrade -b /tmp/backup.tar.gz     # Create backup
sysupgrade -r /tmp/backup.tar.gz     # Restore backup
sysupgrade -v <image>.bin            # Flash a new firmware image (verify path/flags before running!)
```

### Service management

```sh
/etc/init.d/<service> start|stop|restart|reload|enable|disable|status
```

---

## Sources

- OpenWrt 25.12 release announcement and known issues — mailing list post: https://lists.openwrt.org/pipermail/openwrt-announce/2026-March/000081.html
- OpenWrt 25.12.0 coverage (apk transition, ASU, `owut`, hardware support): https://www.helpnetsecurity.com/2026/03/09/openwrt-25-12-0-released/
- OpenWrt 25.12 apk package manager coverage: https://linuxiac.com/openwrt-25-12-released-with-apk-package-manager-replacing-opkg/
- OpenWrt releases page (24.10 EOL timeline, upgrade notes): https://github.com/openwrt/openwrt/releases
- OpenWrt firewall (fw4/nftables) documentation mirror: https://openwrt-openwrt-35.mintlify.app/networking/firewall
- OpenWrt package manager documentation mirror: https://openwrt-openwrt-35.mintlify.app/concepts/package-manager
- opkg→apk cheat sheet reference (community-sourced comparison table): https://forums.raspberrypi.com/viewtopic.php?t=384473
- Bufferbloat.net — "Getting SQM Running Right": https://www.bufferbloat.net/projects/bloat/wiki/Getting_SQM_Running_Right/
- Bufferbloat.net — CAKE reference (incl. CAKE_MQ note for 25.12/Linux 7.0-aligned kernels): https://www.bufferbloat.net/projects/codel/wiki/Cake/
- SQM/flow-offloading conflict discussion — OpenWrt forum: https://forum.openwrt.org/t/how-does-software-flow-offloading-interact-with-sqm/114310
- SQM/QoS & hardware flow offloading experimental-compatibility note — OpenWrt forum: https://forum.openwrt.org/t/sqm-qos-hardware-flow-offloading/214171
- firewall4 architecture/nftables integration reference: https://deepwiki.com/openwrt/firewall4
- WireGuard UCI setup pattern (server, firewall zone) — community write-up: https://casept.github.io/post/wireguard-server-on-openwrt-router/
- WireGuard UCI setup pattern (client/server peers, `[-1]` indexing) — community write-up: https://volatilesystems.org/setting-up-wireguard-on-openwrt.html
- Imou HX21 hardware specifications and OpenWrt support request/merge — OpenWrt forum: https://forum.openwrt.org/t/request-for-openwrt-support-imou-hx21-router-mediatek-mt7981ba/236092
- Imou HX21 / LC-HX3001 hardware specs and board-support merge thread: https://forum.openwrt.org/t/adding-support-for-imou-hx21/242973
- Upstream commit merging Imou HX21 support (`mediatek: add support for Imou HX21`): http://lists.infradead.org/pipermail/lede-commits/2025-November/028095.html
- ImmortalWrt install walkthrough for Imou HX21 / LC-HX3001: https://ffrafat.medium.com/install-immortalwrt-on-imou-hx21-lc-hx3001-easy-openwrt-guide-4cdd95dccf24
- Imou HX21 tweaks/optimization scripts repository: https://github.com/afk901/imou-hx21-openwrt
- ImmortalWrt discussion — HX21 vs LC-HX3001 flashing notes: https://github.com/immortalwrt/immortalwrt/discussions/2085

---

*This guide targets OpenWrt 25.12 on the Imou HX21 (`mediatek/filogic` target). Community support for this specific board is recent (merged late 2025); verify current package availability and any regressions against the forum threads above before relying on this in production. Interface names, exact package names, and firmware behavior may differ depending on your specific build (upstream OpenWrt vs. ImmortalWrt) and network topology — treat every command here as a template to verify against your own router, not a guaranteed copy-paste.*
