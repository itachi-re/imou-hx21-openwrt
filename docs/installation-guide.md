# Installing OpenWrt on the Imou HX21 — Complete Installation & Recovery Guide

<p align="left">
  <img src="https://img.shields.io/badge/OpenWrt-25.12.5-1D2B36?style=for-the-badge&logo=openwrt&logoColor=1DE0B1" alt="OpenWrt 25.12.5">
  <img src="https://img.shields.io/badge/Target-mediatek%2Ffilogic-0D597F?style=for-the-badge" alt="mediatek/filogic target">
  <img src="https://img.shields.io/badge/Board%20support-since%20Nov%202025-yellow?style=for-the-badge" alt="Board support since Nov 2025">
  <br/>
  <img src="https://img.shields.io/badge/risk-bootloader%20flashing%20can%20brick-CC0000?style=flat-square" alt="Risk warning">
  <img src="https://img.shields.io/badge/reversible-yes%2C%20with%20caveats-orange?style=flat-square" alt="Reversible with caveats">
</p>

> **Verification statement:** Every firmware filename, SHA-256 checksum, and download URL in this guide was confirmed directly against `downloads.openwrt.org` for the OpenWrt **25.12.5** stable release (current stable as of writing) on 2026-08-13. The flashing procedure and partition layout below is taken verbatim from the upstream commit that added HX21 support (`2462b36f0cb29fcc88ef45d9ede4bc897dd9e2ba`, merged into `openwrt/openwrt` on 2025-11-20), cross-referenced against the OpenWrt forum threads where owners of the device documented their own installs. Where community sources go further than the official commit message (e.g. specific TFTP client software, config-restore SSH-enable methods), this is explicitly marked **[COMMUNITY-SOURCED, unverified by OpenWrt project]** — treat that material with proportionally more caution.

---

## Table of Contents

1. [Overview](#1-overview)
2. [Hardware Information](#2-hardware-information)
3. [Requirements](#3-requirements)
4. [Before You Start](#4-before-you-start)
5. [Firmware Files](#5-firmware-files)
6. [Backup Stock Firmware](#6-backup-stock-firmware)
7. [Installation Methods](#7-installation-methods)
8. [Linux Method](#8-linux-method)
9. [Windows Method](#9-windows-method)
10. [Android Method](#10-android-method)
11. [First Boot](#11-first-boot)
12. [Verification](#12-verification)
13. [Updating OpenWrt](#13-updating-openwrt)
14. [Returning to Stock](#14-returning-to-stock)
15. [Recovery / Debrick](#15-recovery--debrick)
16. [UART Recovery](#16-uart-recovery)
17. [Troubleshooting](#17-troubleshooting)
18. [Useful Commands](#18-useful-commands)
19. [Safety Checklist](#19-safety-checklist)
20. [References](#20-references)

---

## 1. Overview

**OpenWrt** is a Linux-based, source-buildable firmware distribution for embedded devices — mainly routers — that replaces a vendor's closed firmware with a fully configurable network operating system (LuCI web UI, full shell access, package management, arbitrary routing/firewall/VPN configuration).

**Why OpenWrt is supported on the HX21:** the router uses MediaTek's Filogic 820 (MT7981B) SoC, which OpenWrt's `mediatek/filogic` target already supports broadly. A developer (Jahidul Islam) tore the HX21 down, confirmed it shares its board layout closely with the already-supported **Imou LC-HX3001** (the HX21 is effectively the global-market variant of that Chinese-market device), and submitted board-support patches that were merged upstream on **2025-11-20** ([PR #20753](https://github.com/openwrt/openwrt/pull/20753)).

**Current support status:**
- Board support is **upstream and current** — device tree, U-Boot config, and image definitions live in the mainline `openwrt/openwrt` tree, not a fork or out-of-tree patch set.
- It is included in **OpenWrt 25.12.x** (verified present in 25.12.5's `mediatek/filogic` image directory).
- It is **not** present in OpenWrt 24.10 or earlier — those releases predate the merge.
- **ImmortalWrt** (an OpenWrt fork with a different default package set/theme) has supported the closely related LC-HX3001 for longer, and several community guides use ImmortalWrt images built from LC-HX3001 configs interchangeably with HX21 hardware, since the two devices are hardware-near-identical. This guide focuses on **upstream OpenWrt**, which now has native HX21 support.

**Is there an OEM/easy installation method?** No single-click method is officially documented. The stock Imou firmware does not expose a "flash third-party firmware" option; getting to a flashable state requires either:
- Gaining SSH/root access to the stock firmware first (two methods exist — see [§7](#7-installation-methods)), then using U-Boot's TFTP recovery mode, or
- Physical UART access if the SSH-access methods don't work on your unit's firmware revision.

**What gets replaced:** the entire writable firmware stack — U-Boot's **BL2** (preloader) and **FIP** (BL31 + main U-Boot) stages, and the **UBI** partition holding the kernel/rootfs. The **`factory`** MTD partition (containing calibration data and the router's MAC address) is deliberately **left untouched** by the standard procedure — do not erase or overwrite it.

**Is installation reversible?** Partially. Returning to 100% stock Imou firmware is not officially documented or supported by the OpenWrt project — see [§14](#14-returning-to-stock) for what is and isn't realistic. Within OpenWrt/ImmortalWrt itself, moving between versions via `sysupgrade` is fully reversible and safe.

> ⚠️ **This cannot be stated strongly enough: flashing the BL2 (preloader) or FIP partitions incorrectly — wrong file, interrupted write, wrong `mtd` target — can permanently brick the device.** Unlike a bad `sysupgrade` (recoverable via U-Boot's TFTP recovery, since U-Boot itself survives), a bad BL2/FIP write can leave the router unable to boot far enough to even enter recovery mode, at which point only a UART-based reflash (requiring you to open the case and solder/clip onto exposed pads) can save it — and even that is not guaranteed. Read [§19](#19-safety-checklist) before touching `mtd write` on `bl2` or `fip`.

---

## 2. Hardware Information

| Property | Value | Source |
|---|---|---|
| Device | Imou HX21 (global variant of Imou **LC-HX3001**) | Upstream commit, forum thread |
| SoC | MediaTek MT7981B "Filogic 820", dual-core ARM Cortex-A53 | Upstream commit |
| CPU clock | ~1.3 GHz | Community teardown (forum) |
| RAM | 256 MB DDR3 | Upstream commit |
| Flash (NAND) | 128 MB SPI-NAND, **Foresee F35SQA001G** | Upstream commit |
| Ethernet | 4× 10/100/1000 Mbps | Upstream commit |
| Switch IC | MediaTek MT7531AE | Upstream commit |
| Wi-Fi | MediaTek MT7976C, Wi-Fi 6 (AX3000-class, dual-band) | Upstream commit |
| Buttons | Reset, Mesh | Upstream commit |
| Power | DC 12 V / 1 A | Upstream commit |
| Bootloader | U-Boot (MediaTek fork), with BL2 preloader + BL31/FIP stage | Upstream `uboot-mediatek` patch |
| UART | Present; **not** trivially accessible on all units — pads may require opening the case / soldering | Forum thread (ImmortalWrt discussion #2085) |
| Serial settings | **115200 8N1** (115200 baud, 8 data bits, no parity, 1 stop bit) | Device tree (`stdout-path = "serial0:115200n8"`) |
| OpenWrt target | `mediatek` | Upstream commit |
| OpenWrt subtarget | `filogic` | Upstream commit |
| OpenWrt device ID | `imou_hx21` | `filogic.mk` |
| Architecture | `aarch64_cortex-a53` (ARM64) | Standard for `mediatek/filogic` |
| Supported since | OpenWrt commit merged 2025-11-20; first appears in stable release **25.12** | Upstream commit + downloads.openwrt.org |
| Current stable OpenWrt release | **25.12.5** (released 2026-06-29) | downloads.openwrt.org |

### MTD partition layout (from the device tree)

| Partition | Offset | Size | Notes |
|---|---|---|---|
| `bl2` | `0x000000` | 1 MB | Preloader. Read-only in normal operation; **do not touch unless you know exactly why**. |
| `u-boot-env` | `0x100000` | 512 KB | U-Boot environment variables. |
| `factory` | `0x180000` | 2 MB | Calibration data + MAC address. **Never erase.** |
| `fip` | `0x380000` | 2 MB | BL31 + main U-Boot (FIP image). |
| `ubi` | `0x580000` | remainder (~123 MB) | UBI volume holding OpenWrt's kernel/rootfs (`fit`), recovery image (`recovery`), U-Boot env redundancy (`ubootenv`/`ubootenv2`), and overlay (`rootfs_data`). |

Knowing this layout matters for [§15](#15-recovery--debrick) and for understanding exactly what a "backup everything, especially Factory" instruction is protecting.

---

## 3. Requirements

### Linux

| Tool | Purpose | Arch | openSUSE | Fedora | Debian/Ubuntu |
|---|---|---|---|---|---|
| SSH client | Access router shell | `openssh` (base) | `openssh` (base) | `openssh-clients` | `openssh-client` |
| SCP | Transfer files to/from router | bundled with `openssh` | bundled | bundled | bundled |
| TFTP server | Serve recovery/initramfs image to U-Boot | `tftp-hpa` (`atftpd` alt.) | `pkg install tftp-server` | `dnf install tftp-server` | `apt install tftpd-hpa` |
| `mtd` tools (only needed if working *inside* OpenWrt shell, not on your PC) | Low-level flash writes | pre-installed on OpenWrt images | n/a | n/a | n/a |
| `gzip` | Decompress firmware archives if needed | usually pre-installed | usually pre-installed | usually pre-installed | usually pre-installed |
| `ip` (`iproute2`) | Configure static IP on your PC's NIC | `pacman -S iproute2` | `zypper install iproute2` | `dnf install iproute2` | `apt install iproute2` |
| `curl` / `wget` | Download firmware images | `pacman -S curl wget` | `zypper install curl wget` | `dnf install curl wget` | `apt install curl wget` |
| Serial terminal (only for UART work) | Console over UART adapter | `picocom`/`minicom` via `pacman` | via `zypper` | via `dnf` | via `apt` |

```sh
# Arch
sudo pacman -S openssh tftp-hpa iproute2 curl wget picocom

# openSUSE (your daily driver, itachi)
sudo zypper install openssh tftp iproute2 curl wget picocom

# Fedora
sudo dnf install openssh-clients tftp-server iproute curl wget picocom

# Debian / Ubuntu
sudo apt install openssh-client tftpd-hpa iproute2 curl wget picocom
```

### Windows

| Tool | Purpose |
|---|---|
| **PuTTY** or Windows Terminal + OpenSSH (built into Win10/11) | SSH access |
| **WinSCP** | SCP/SFTP file transfer with a GUI |
| **Tftpd64** (or Tftpd32) | TFTP server for serving the recovery image — this is the tool referenced in every community HX21 walkthrough found during research |
| Serial terminal (only for UART work) — PuTTY (serial mode) or **Termite** | Console over UART adapter |

Verified necessity: yes — every Windows-based community walkthrough found (Turkish-language forum guides on Technopat/Techolay, the Medium ImmortalWrt guide) uses exactly this combination (WinSCP + Tftpd64), so this is a well-trodden, low-risk toolchain for this specific device.

### Android

**Investigated and verified realistic scope**, not assumed:

| Step | Feasible on Android (Termux)? | Why / caveat |
|---|---|---|
| SSH/SCP to a router already running OpenWrt (e.g. for `sysupgrade`, ongoing management) | **Yes** | `pkg install openssh` in Termux gives a full `ssh`/`scp` client with no special privileges needed — this part works fine, non-rooted. |
| Serving the TFTP recovery image to U-Boot (the actual "unbrick"/initial-flash step) | **Not recommended / not practical on a non-rooted device** | Standard TFTP servers bind to UDP port 69, which is a privileged port (<1024) requiring root on Linux (Termux's kernel is the phone's Linux kernel). A stock, non-rooted Android/Termux install cannot bind port 69 without root. |
| USB-Ethernet OTG adapter to get a wired link to the router while it's in TFTP-recovery mode | **Possible in principle, practically fragile** | Requires a USB-C/OTG-to-Ethernet adapter your phone recognizes, Termux (or Android's own network settings) able to assign a static IP to it, and — again — root for the TFTP server itself. Even rooted, this is materially more fragile than a laptop: Android's network stack actively manages interfaces and can interfere with a manually-configured static-IP link in ways a Linux/Windows desktop won't. |
| Serial/UART console from Android | **Not practical for this guide's scope** | Requires a USB-serial UART adapter, an OTG cable, and a serial terminal app (a few exist, e.g. "Serial USB Terminal") — feasible for basic bring-up but has not been verified against this specific board's UART pinout by community sources found during research. |

**Bottom line:** if your HX21 already has SSH access via Method 2 (§7) and you just need to push a `sysupgrade` file or do ongoing config, Android/Termux works fine. **The initial bootloader-flash / TFTP-recovery stage of this guide should be done from a Linux or Windows PC with a wired Ethernet port** — that is the honest, verified answer rather than a workaround.

---

## 4. Before Installation

- **Connect via Ethernet, not Wi-Fi.** The entire flashing procedure depends on a stable, low-latency link and (for TFTP) precise static-IP addressing that Wi-Fi's DHCP/roaming behavior can silently interfere with. A dropped Wi-Fi association mid-write is exactly the scenario that bricks devices.
- **Disconnect unnecessary network interfaces** on your PC (other Wi-Fi adapters, VPN tunnels, virtual adapters) so there's no ambiguity about which interface is talking to the router.
- **Set a static IP** on the PC's Ethernet NIC matching what U-Boot expects: `192.168.1.254/24`, gateway `192.168.1.1` (this exact addressing comes directly from the upstream commit's flash instructions — U-Boot's `serverip` env var on this device defaults to `192.168.1.254`, `ipaddr` to `192.168.1.1`).

```sh
# Linux — replace eth0 with your actual interface name (check with `ip -brief link`)
sudo ip addr add 192.168.1.254/24 dev eth0
sudo ip link set eth0 up
ip addr show eth0   # confirm
```

On Windows: Control Panel → Network and Sharing Center → Change adapter settings → right-click your Ethernet adapter → Properties → IPv4 → Use the following IP address → `192.168.1.254`, subnet `255.255.255.0`, leave gateway blank or set `192.168.1.1`.

- **Confirm the PC can reach the router** once it's in the expected state (i.e., after the SSH-access step, or once U-Boot is waiting in TFTP mode):
```sh
ping -c 4 192.168.1.1
```
- **Disable VPNs** that could route traffic away from the local subnet or interfere with the TFTP UDP session.
- **Check firewall rules** on your PC — a local firewall blocking inbound/outbound UDP port 69 (TFTP) will silently break the recovery step with no useful error on the router side. Temporarily allow it, or disable the firewall for the duration of the flash.
- **Verify firmware checksums** — see [§5](#5-firmware-files). Do this before writing anything to flash, not after something goes wrong.
- **Ensure stable power** — direct wall power, no power-strip surge/UPS quirks, no risk of someone unplugging it. A power loss during a `mtd write` to `bl2` or `fip` is one of the few genuinely unrecoverable failure modes for this device.
- **Do not interrupt power during flashing** — this bears repeating because it's the single most common cause of a hard brick in the community threads reviewed for this guide.

---

## 5. Firmware Files

All files below are for **OpenWrt 25.12.5**, the current stable release, confirmed present at `downloads.openwrt.org` at time of writing. Base URL:

```
https://downloads.openwrt.org/releases/25.12.5/targets/mediatek/filogic/
```

| File | Purpose | SHA-256 | Size |
|---|---|---|---|
| `openwrt-25.12.5-mediatek-filogic-imou_hx21-bl31-uboot.fip` | FIP image (BL31 + main U-Boot) — written to the `fip` MTD partition | `6b203411e71692343518c5b97e54c48f4a4c3bfc7925b376aafc5b149cbf598b` | 785.7 KB |
| `openwrt-25.12.5-mediatek-filogic-imou_hx21-preloader.bin` | BL2 preloader — written to the `bl2` MTD partition | `ad13725b82566ebd6b6a0cc37b83a0b9f17b8685702f32efc02fefff4126b1fc` | 224.8 KB |
| `openwrt-25.12.5-mediatek-filogic-imou_hx21-initramfs-recovery.itb` | Initramfs recovery image — served over TFTP, boots into a RAM-resident OpenWrt used to perform the real install | `27b296ae93d86bc3d84d3c15cdb45ec0e30e21088b5642421dbccefddb433eec` | 8896.0 KB |
| `openwrt-25.12.5-mediatek-filogic-imou_hx21-squashfs-sysupgrade.itb` | The actual installable OpenWrt image — written via `sysupgrade` once you're running the initramfs recovery image | `4e6dff2db0ae3e497b1d82f534458d9240ac989c67365ba4082811a47123a247` | 10712.3 KB |

> **Always re-verify these checksums against the live page before flashing**, especially if you're reading this guide any significant time after it was written — point releases (25.12.6, 26.x, etc.) will have different files and different hashes. Do not assume the numbers above stay valid indefinitely.

```sh
sha256sum openwrt-25.12.5-mediatek-filogic-imou_hx21-*.{fip,bin,itb}
# Compare each line against the table above (or against the sha256sum file OpenWrt
# publishes alongside the images) before proceeding.
```

**Snapshot builds** (`https://downloads.openwrt.org/snapshots/targets/mediatek/filogic/`) also carry `imou_hx21` images, built nightly from the unreleased development branch. **Do not use snapshot builds for a production router** — they are explicitly unsupported, untested, and can contain regressions; they exist for testing very recent fixes, not day-to-day use. This guide uses stable 25.12.5 throughout.

**OpenWrt 24.10 does not have HX21 support** (the board-support merge postdates that branch). If you see a community guide referencing 24.10 for this device, it is either using **ImmortalWrt's** separately-maintained LC-HX3001 build (different project, different image names — e.g. `immortalwrt-24.10.4-mediatek-filogic-imou_lc-hx3001-*`) or is simply outdated. Don't mix ImmortalWrt LC-HX3001 files with upstream OpenWrt HX21 files or vice versa — the partition handling and U-Boot environment differ enough between the two projects' builds that this is not a safe interchange to make casually.

---

## 6. Backup Stock Firmware

Once you have SSH access to the stock firmware (§7), back up every MTD partition before writing anything:

```sh
# Run this FROM the stock firmware's SSH shell, not your PC
cat /proc/mtd
# Note the mtdN numbers — they may not match the "bl2/u-boot-env/factory/fip/ubi"
# names 1:1 on stock firmware, since Imou's own partition table may differ from
# OpenWrt's. Identify each by size against the table in §2.

for part in $(cat /proc/mtd | tail -n +2 | cut -d: -f1); do
  dd if=/dev/$part of=/tmp/${part}.bin
done

# Copy everything off the router before proceeding
scp root@192.168.1.1:/tmp/mtd*.bin ./hx21-stock-backup/
```

**Specifically prioritize backing up the `factory` partition** — it's the one holding your device's unique calibration data and MAC address, which cannot be regenerated if lost, and which the standard OpenWrt flash procedure is designed to never touch (so you shouldn't need this backup in the normal case — but "shouldn't need" is not "can't lose to a mistake").

If your PC's `/tmp` fills up or the router's overlay is too small to hold all the dumps simultaneously, back up and offload one partition at a time instead of trying to hold copies of everything on-device at once — 128 MB of total NAND doesn't leave much slack for temporary duplicate copies of itself.

---

## 7. Installation Methods

Getting from stock Imou firmware to a flashable state requires SSH access to the stock firmware first. Two methods are documented in the upstream OpenWrt commit itself:

### Method 1 — UART (physical access required)

1. Open the case and connect a 3.3V UART adapter to the router's UART pads (115200 8N1 — see [§2](#2-hardware-information) and [§16](#16-uart-recovery)).
2. The UART console is accessible on stock firmware; use it to set a root password (`passwd`) and manually start the dropbear SSH daemon on port 22.
3. This gives you a normal SSH session at whatever IP the stock firmware has DHCP'd or is statically configured to.

This is the officially-documented, most reliable method, but requires opening the case — on some HX21 units the UART pads are not easily reachable without desoldering a shield or component, per owner reports in the ImmortalWrt discussion thread.

### Method 2 — Config-restore SSH enable (no disassembly)

1. Log into the stock web interface (commonly `http://192.168.10.1`, credentials `admin/admin` on units investigated by the community — **verify your own unit's actual default IP/credentials first**, as OEM firmware defaults can vary by batch/region).
2. Navigate to **System → Backup/Restore → Restore Configuration**, and upload a specially-crafted configuration file that enables SSH on restore.
3. After reboot: web UI password becomes `12345678`, SSH is open with username `root` and **no password**.

**[COMMUNITY-SOURCED, unverified by OpenWrt project]** — the *existence* and general mechanism of this method (config-restore enabling SSH) is stated directly in the upstream commit message as "Method 2," but the upstream commit does not itself distribute a specific config file — it assumes you have one or can construct one. Community forum threads reference a specific pre-built config file hosted on a third-party file-sharing link for this exact purpose. **This guide deliberately does not link to or endorse a specific third-party binary/config file for this step** — flashing an unverified binary onto your router's bootloader-adjacent config store is exactly the kind of supply-chain risk this guide is trying to help you avoid elsewhere. If you go this route, get the file from a source you can audit or from someone in the OpenWrt/ImmortalWrt community whose identity and track record you can verify, and treat "just trust this random config file" as the highest-risk step in the entire process — higher than the bootloader flash itself, because at least the bootloader flash comes with official checksums.

If neither method works cleanly on your unit's specific firmware revision, the OpenWrt and ImmortalWrt forum threads linked in [§20](#20-references) are the right place to check for revision-specific notes before assuming your only option is opening the case.

---

## 8. Linux Method

Once you have SSH access to stock firmware (§7) and have backed it up (§6):

**1. Download and verify the firmware files** (§5):
```sh
mkdir -p ~/hx21-flash && cd ~/hx21-flash
BASE=https://downloads.openwrt.org/releases/25.12.5/targets/mediatek/filogic
for f in bl31-uboot.fip preloader.bin initramfs-recovery.itb squashfs-sysupgrade.itb; do
  curl -LO "${BASE}/openwrt-25.12.5-mediatek-filogic-imou_hx21-${f}"
done
sha256sum openwrt-25.12.5-mediatek-filogic-imou_hx21-*  # compare against §5 table
```

**2. Set your PC's static IP** (§4) and copy the FIP image to the router:
```sh
scp openwrt-25.12.5-mediatek-filogic-imou_hx21-bl31-uboot.fip root@192.168.1.1:/tmp/
```

**3. Write the new FIP** — run this **on the router**, over SSH:
```sh
mtd write /tmp/openwrt-25.12.5-mediatek-filogic-imou_hx21-bl31-uboot.fip FIP
```
> This is the point of no return for the *stock bootloader's second stage*. Confirm your file's checksum one more time before this line.

**4. Set up TFTP to serve the recovery image**, on your PC:
```sh
# tftpd-hpa (Debian/Ubuntu) — edit /etc/default/tftpd-hpa:
#   TFTP_DIRECTORY="/srv/tftp"
#   TFTP_ADDRESS="192.168.1.254:69"
sudo mkdir -p /srv/tftp
sudo cp openwrt-25.12.5-mediatek-filogic-imou_hx21-initramfs-recovery.itb /srv/tftp/
sudo systemctl restart tftpd-hpa
```
The **exact filename matters** — U-Boot's environment on this device requests `openwrt-mediatek-filogic-imou_hx21-initramfs-recovery.itb` (per the `defenvs/imou_hx21_env` bootfile variable in the upstream patch) — some community guides rename the downloaded file to drop the version number prefix to match. Check your specific build's environment (`printenv bootfile` at the U-Boot prompt over UART, if you have it) rather than assuming; if unsure, serve it under both the versioned and unversioned filename.

**5. Power-cycle the router and let it pull the recovery image over TFTP.** With BL2/FIP already updated to OpenWrt's U-Boot, the device should enter TFTP recovery automatically or via the boot menu (accessible via UART, or automatically on first boot after a FIP write per the `_firstboot` env logic in the upstream patch). Watch your TFTP server's logs:
```sh
sudo journalctl -u tftpd-hpa -f
```
Wait for the transfer to complete and the router to boot the initramfs image (a couple of minutes).

**6. Confirm you can reach the initramfs-booted OpenWrt**, then push the real sysupgrade:
```sh
ping -c 4 192.168.1.1
scp openwrt-25.12.5-mediatek-filogic-imou_hx21-squashfs-sysupgrade.itb root@192.168.1.1:/tmp/
ssh root@192.168.1.1 'sysupgrade -n /tmp/openwrt-25.12.5-mediatek-filogic-imou_hx21-squashfs-sysupgrade.itb'
```
`-n` discards any settings picked up during the transient initramfs boot rather than trying to carry them into the installed system — appropriate for a first install.

**7. (Only if you also need to update BL2)** — per the upstream commit, this is a separate, later step performed *after* OpenWrt is fully installed and booted, not during initial install:
```sh
# From an SSH session into the now-installed OpenWrt (apk-based, 25.12+):
apk update && apk add kmod-mtd-rw
insmod mtd-rw i_want_a_brick=1
mtd write /tmp/openwrt-25.12.5-mediatek-filogic-imou_hx21-preloader.bin bl2
reboot
```
The `i_want_a_brick=1` parameter is `mtd-rw`'s own explicit, deliberately-alarming confirmation flag for enabling writes to normally-read-only MTD partitions — treat it as the tool's way of making you pause here, not as decoration to skip past.

---

## 9. Windows Method

**1. Download and verify firmware files** (§5) via browser; verify checksums with PowerShell:
```powershell
Get-FileHash .\openwrt-25.12.5-mediatek-filogic-imou_hx21-bl31-uboot.fip -Algorithm SHA256
# Compare output against the table in §5
```

**2. Set a static IP** on your Ethernet adapter (§4, Windows instructions).

**3. Upload the FIP image with WinSCP**: connect to `root@192.168.1.1`, drag `openwrt-25.12.5-mediatek-filogic-imou_hx21-bl31-uboot.fip` into `/tmp/`.

**4. Write the FIP** via PuTTY (or Windows Terminal's `ssh`) session to the router:
```sh
mtd write /tmp/openwrt-25.12.5-mediatek-filogic-imou_hx21-bl31-uboot.fip FIP
```

**5. Configure Tftpd64**:
   - **Current Directory**: the folder containing your downloaded recovery `.itb` file.
   - **Server interfaces**: select `192.168.1.254` (the static IP you configured).
   - Rename `openwrt-25.12.5-mediatek-filogic-imou_hx21-initramfs-recovery.itb` to match whatever filename your unit's U-Boot environment requests (see the filename note in §8 step 4) — community guides for this exact device typically drop the version segment, e.g. `openwrt-mediatek-filogic-imou_hx21-initramfs-recovery.itb`.

**6. Power-cycle the router.** Watch Tftpd64's **Log Viewer** pane for the transfer — you should see a `RRQ` request for your recovery filename followed by data blocks. If nothing appears, see [§17](#17-troubleshooting) (this exact stuck-at-TFTP symptom is the single most common problem reported in the Turkish-language community threads reviewed for this guide).

**7. Once booted into the recovery image**, use WinSCP to upload `openwrt-25.12.5-mediatek-filogic-imou_hx21-squashfs-sysupgrade.itb` to `/tmp/` on the router, then over SSH/PuTTY:
```sh
sysupgrade -n /tmp/openwrt-25.12.5-mediatek-filogic-imou_hx21-squashfs-sysupgrade.itb
```

**8. BL2 update (optional, after OpenWrt is running)** — same commands as the Linux method's step 7, run from PuTTY connected to the now-installed OpenWrt system.

---

## 10. Android Method

As established in [§3](#3-requirements), **the initial flash should not be attempted from Android** — a non-rooted phone cannot reliably serve the TFTP recovery step (privileged port 69), and even rooted phones with USB-Ethernet OTG add fragility that isn't worth the risk on a step where a failure is expensive.

**What Android/Termux is genuinely good for, once OpenWrt is already installed:**

```sh
pkg install openssh
ssh root@192.168.1.1
scp openwrt-25.12.5-mediatek-filogic-imou_hx21-squashfs-sysupgrade.itb root@192.168.1.1:/tmp/
```

Ongoing management (LuCI over a mobile browser, SSH-based config, pushing future `sysupgrade` files for point releases) all work fine from Android with zero caveats. It's specifically the **bootloader-flash / TFTP-recovery bring-up** that this guide can't responsibly hand you a "just do it from your phone" procedure for.

---

## 11. First Boot

After `sysupgrade` completes and the router reboots into the newly-installed system:

- Default LAN IP: **`192.168.1.1`** (standard OpenWrt default; confirm this matches what you observe — device-tree/env overrides are possible but none were found for this device).
- LuCI web UI: `http://192.168.1.1/` — first boot has **no root password set**; set one immediately (`passwd` over SSH, or via LuCI's System → Administration page) before exposing the router to any untrusted network.
- SSH: `ssh root@192.168.1.1` — works immediately, no password, until you set one.
- Wi-Fi radios ship **disabled by default** on a fresh OpenWrt install (this is standard OpenWrt behavior, not something specific to this device) — enable via LuCI's **Network → Wireless** page or:
```sh
uci set wireless.radio0.disabled='0'
uci set wireless.radio1.disabled='0'
uci commit wireless
wifi reload
```

---

## 12. Verification

Confirm you're actually running what you think you're running:

```sh
cat /etc/openwrt_release
# Should show DISTRIB_RELEASE='25.12.5' and DISTRIB_TARGET='mediatek/filogic'

ubus call system board
# Confirms board name, "imou,hx21" compatible string, kernel version

apk --version
# Confirms apk is present (25.12+ package manager)

df -h /overlay
# Sanity-check available flash headroom before installing extra packages
```

---

## 13. Updating OpenWrt

For point releases within the same major branch (e.g. 25.12.5 → 25.12.6, once it exists), a normal `sysupgrade` with `-c` (retain config) is sufficient — no bootloader re-flash needed, since only the `ubi` partition's kernel/rootfs volume changes:

```sh
sysupgrade -c /path/to/openwrt-25.12.X-mediatek-filogic-imou_hx21-squashfs-sysupgrade.itb
```

Always re-download and re-verify the sysupgrade image's checksum for the *specific* point release you're moving to — don't reuse an old file. For a major version bump (25.12 → the next major series, whenever that ships), read that release's own upgrade notes first; major-version jumps occasionally carry partition-layout or U-Boot-environment changes that point releases don't.

---

## 14. Returning to Stock

**This is not officially documented or supported by the OpenWrt project.** Realistic options, roughly in order of how intact your original backup needs to be:

1. **Full MTD restore from your §6 backup**, partition by partition, via `mtd write` for each dumped image — requires you actually made a complete backup *before* flashing, and requires the restore procedure itself to not hit the same brick risks as the original install (bad write, power loss).
2. **Manufacturer recovery tools**, if Imou publishes any — none were found during research for this guide; if Imou has an official recovery utility, it wasn't surfaced by the sources checked here, so don't assume one exists without checking directly with Imou support.
3. **Accept that "stock" isn't coming back cleanly** and instead treat ImmortalWrt/OpenWrt as the router's permanent OS — this is the de facto outcome for most community members in the threads reviewed, several of whom note there wasn't a clean documented path back to stock that they found either.

If returning to stock firmware is a hard requirement for you (e.g. warranty service), **do not proceed with this guide** until you've confirmed with Imou support what recovery options they provide, since this guide cannot respons­ibly promise a path back that wasn't found to exist in the sources checked.

---

## 15. Recovery / Debrick

If something goes wrong **short of a full BL2/FIP failure**, U-Boot's own recovery mechanics (from the `defenvs/imou_hx21_env` shipped in the upstream patch) give you several soft-recovery paths, accessible via the boot menu:

| Boot menu option | What it does |
|---|---|
| `1` — Boot system via TFTP | Boots a TFTP-served image without writing it to flash — the safe way to test an image first. |
| `2` / `3` — Boot production/recovery system from NAND | Boots whichever OpenWrt copy is currently in the `fit`/`recovery` UBI volumes. |
| `4` / `5` — Load production/recovery via TFTP **then write to NAND** | The actual install/reinstall path — this is what §8/§9 walk through implicitly via the automatic first-boot flow. |
| `6` — Load BL31+U-Boot FIP via TFTP then write to NAND | Re-flash the FIP partition (shown in red in the boot menu — a deliberate visual warning baked into the environment itself). |
| `7` — Load BL2 preloader via TFTP then write to NAND | Re-flash BL2 (also shown in red). |
| `9` — Reset all settings to factory defaults | Resets U-Boot env, not the OpenWrt install itself. |

**Accessing this boot menu requires either UART access, or — per the environment's own `boot_first` logic — holding the reset button during power-on**, which triggers `boot_tftp_recovery` automatically (`if button reset ; then ... run boot_tftp_recovery`). This is genuinely useful: **holding reset while powering on and having a TFTP server ready with the recovery `.itb` is your primary recovery path for a bad OpenWrt install that doesn't involve a corrupted bootloader.**

```sh
# Hold the physical reset button, power on, keep holding for several seconds,
# release once the router starts requesting the recovery image over TFTP
# (watch your TFTP server logs to confirm the RRQ arrives).
```

**A bricked BL2 or FIP is a different situation** — the device may not get far enough to run any of this U-Boot environment logic at all, in which case only [UART recovery](#16-uart-recovery) (or, if UART itself doesn't get you a working console, a full chip-level reflash that's out of scope for this guide) can help.

---

## 16. UART Recovery

**Serial settings:** 115200 baud, 8 data bits, no parity, 1 stop bit (**115200 8N1**) — taken directly from the device tree's `stdout-path = "serial0:115200n8"`.

**What you need:**
- A 3.3V-logic USB-to-UART adapter (**not** a 5V adapter — 5V logic on 3.3V-rated GPIO pins can damage the SoC).
- Physical access to the board's UART pads (TX, RX, GND — VCC pad, if present, should generally **not** be connected, since the router's own power supply already powers the board; connecting adapter VCC as well risks a voltage conflict).
- A serial terminal: `picocom`/`minicom` (Linux), PuTTY/Termite (Windows).

```sh
# Linux, once the adapter shows up as e.g. /dev/ttyUSB0
sudo picocom -b 115200 /dev/ttyUSB0
```

From here you get a live U-Boot console on power-up (interrupt autoboot to reach the menu), and — critically — a way to attempt `mtd`-level recovery even if the network-facing recovery paths in [§15](#15-recovery--debrick) are unavailable because BL2/FIP is damaged, since U-Boot's serial console doesn't depend on those stages being fully healthy the same way network boot does. The precise steps for a from-scratch BL2/FIP reflash via UART depend on how badly damaged the current bootloader state is — beyond generic guidance, this is genuinely a "post in the OpenWrt forum thread with your specific UART log output" situation, since debrick procedures are necessarily case-by-case.

**Note on pad accessibility:** per the ImmortalWrt discussion thread, UART pads on at least some HX21 units are **not** easily reachable without opening the case and doing some amount of soldering/probing — this isn't a "just clip a header on" job on every unit. Factor that into your risk tolerance before starting a network-based flash on a unit where UART is not already confirmed accessible as your fallback.

---

## 17. Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| TFTP never receives a request from the router ("modem never pulled data," per multiple community reports) | Recovery filename mismatch, wrong static IP/subnet, PC firewall blocking UDP/69, or router isn't actually in TFTP-wait state | Confirm filename matches what U-Boot's env requests exactly (§8 step 4); confirm `192.168.1.254/24` is actually applied (`ip addr show`); temporarily disable PC firewall; confirm via UART (if available) that the router is at the TFTP-wait prompt at all |
| TFTP transfer starts but times out partway | Flaky Ethernet cable/port, or PC-side TFTP server settings (block size, timeout) misconfigured | Try a different cable/port; for `tftpd-hpa`, confirm `TFTP_OPTIONS="--secure"` isn't blocking access outside the served directory in an unexpected way |
| Router unreachable at `192.168.1.1` after what looked like a successful flash | Router may have a different LAN IP configured, or is still mid-boot (first boot after a UBI write can take noticeably longer than subsequent boots) | Wait 2–3 minutes; check your PC's ARP table (`ip neigh`) for any device response; if you have UART, watch the actual boot log |
| SSH to stock firmware refuses connection even after "Method 2" config restore | Config file didn't match your specific firmware revision, or restore didn't actually apply | Re-check the exact stock firmware version/build against whatever the config file was built for; consider Method 1 (UART) instead |
| `mtd write ... bl2` refuses / device not found | `mtd-rw` module not loaded, or `i_want_a_brick=1` param omitted | Re-run `insmod mtd-rw i_want_a_brick=1` exactly; confirm with `cat /proc/mtd` that a `bl2` partition is visible before writing |
| Wi-Fi doesn't appear after first boot | Radios ship disabled by default on stock OpenWrt | Enable via LuCI or `uci`/`wifi reload` — see [§11](#11-first-boot) |
| Internet doesn't work despite LAN/DHCP working (reported in Turkish community thread) | WAN interface/protocol not configured for actual ISP connection type (PPPoE vs DHCP vs static, VLAN tagging requirements) | Configure **Network → Interfaces → WAN** to match what your ISP actually requires — this is unrelated to the HX21 port itself and is standard first-router-setup configuration |
| Device appears completely dead — no LEDs, no network response at all | Possible BL2/FIP-level brick, or (less dramatically) just needs longer to boot / a stuck failed U-Boot loop | Attempt hold-reset-on-power-on TFTP recovery first (§15); if genuinely no response at all, move to UART (§16) to get visibility into what state it's actually in before assuming worst-case |

---

## 18. Useful Commands

### On your PC (Linux)
```sh
ip addr                                  # confirm static IP applied
ip route                                 # confirm no conflicting routes
ping -c 4 192.168.1.1                    # reachability check
sha256sum <file>                         # verify firmware checksum
sudo journalctl -u tftpd-hpa -f          # live TFTP server log
scp <file> root@192.168.1.1:/tmp/        # push a file to the router
ssh root@192.168.1.1                     # shell into the router
```

### On the router (OpenWrt, apk-based 25.12+)
```sh
cat /proc/mtd                            # list MTD partitions
cat /etc/openwrt_release                 # confirm installed version/target
ubus call system board                   # board identity, compatible string
apk update && apk add <pkg>              # package management
sysupgrade -n <image>.itb                # install without keeping settings
sysupgrade -c <image>.itb                # install, keep settings (point releases)
mtd write <file> <partition>             # low-level flash write — use with care
insmod mtd-rw i_want_a_brick=1           # enable writes to normally-RO MTD parts
reboot                                   # reboot
```

### On the router (U-Boot console, via UART)
```
printenv                 # dump full U-Boot environment
printenv bootfile        # confirm exact TFTP filename U-Boot expects
run boot_tftp_recovery   # manually trigger TFTP recovery load+write
bootmenu                 # re-enter the interactive boot menu
```

---

## 19. Safety Checklist

Before running **any** `mtd write` targeting `bl2` or `fip`:

- [ ] I have verified the SHA-256 checksum of every file against the live `downloads.openwrt.org` page, not an old cached value.
- [ ] I am on a **wired** Ethernet connection, not Wi-Fi.
- [ ] My PC's static IP (`192.168.1.254/24`) is confirmed applied (`ip addr show` / `ipconfig`).
- [ ] The router is on stable, direct wall power — no risk of accidental disconnection.
- [ ] I have backed up all stock MTD partitions (§6), especially `factory`, and copied the backups off the router.
- [ ] I have a UART fallback plan **or** have explicitly accepted the risk of not having one (i.e., I've confirmed my unit's UART pads are/aren't accessible before starting, not after something goes wrong).
- [ ] I am flashing the `imou_hx21`-specific files, not LC-HX3001/ImmortalWrt files or another device's files.
- [ ] I am not interrupting power at any point during a `bl2`/`fip` write, even if it appears to hang — I will wait, and only investigate via a second channel (ping, TFTP logs, UART if available) rather than power-cycling mid-write.
- [ ] I understand returning to fully-stock Imou firmware is not a guaranteed, officially-supported path (§14), and I've decided that's an acceptable tradeoff.

---

## 20. References

**Official OpenWrt sources:**
- Upstream commit adding HX21 support (full patch, partition layout, U-Boot env, flash instructions) — [lists.infradead.org/pipermail/lede-commits/2025-November/028095.html](http://lists.infradead.org/pipermail/lede-commits/2025-November/028095.html)
- Merged pull request — [github.com/openwrt/openwrt/pull/20753](https://github.com/openwrt/openwrt/pull/20753)
- OpenWrt 25.12.5 firmware directory (mediatek/filogic target) — [downloads.openwrt.org/releases/25.12.5/targets/mediatek/filogic/](https://downloads.openwrt.org/releases/25.12.5/targets/mediatek/filogic/)
- OpenWrt downloads index (current stable release pointer) — [downloads.openwrt.org](https://downloads.openwrt.org/)
- OpenWrt releases / changelogs — [github.com/openwrt/openwrt/releases](https://github.com/openwrt/openwrt/releases)

**Community sources (owner-reported, not OpenWrt-official):**
- OpenWrt forum — "Adding Support For Imou HX21" (developer's own hardware writeup and thread) — [forum.openwrt.org/t/adding-support-for-imou-hx21/242973](https://forum.openwrt.org/t/adding-support-for-imou-hx21/242973)
- OpenWrt forum — "Request for OpenWrt Support: Imou HX21 Router (MediaTek MT7981BA)" (original hardware teardown request) — [forum.openwrt.org/t/request-for-openwrt-support-imou-hx21-router-mediatek-mt7981ba/236092](https://forum.openwrt.org/t/request-for-openwrt-support-imou-hx21-router-mediatek-mt7981ba/236092)
- ImmortalWrt GitHub discussion — HX21 vs LC-HX3001, SSH-access methods, UART accessibility notes — [github.com/immortalwrt/immortalwrt/discussions/2085](https://github.com/immortalwrt/immortalwrt/discussions/2085)
- ImmortalWrt GitHub issue — original HX21/LC-HX3001 similarity request — [github.com/immortalwrt/immortalwrt/issues/2084](https://github.com/immortalwrt/immortalwrt/issues/2084)
- Medium — "Install ImmortalWrt on Imou HX21 Router: LC-HX3001 Easy OpenWrt Guide" (Windows/WinSCP/Tftpd64 walkthrough, ImmortalWrt-specific) — [ffrafat.medium.com](https://ffrafat.medium.com/install-immortalwrt-on-imou-hx21-lc-hx3001-easy-openwrt-guide-4cdd95dccf24)
- Technopat Sosyal (Turkish) — HX21 OpenWrt install walkthrough, TFTP troubleshooting reports — [technopat.net/sosyal/konu/imou-hx21-openwrt-kurulumu.4034814](https://www.technopat.net/sosyal/konu/imou-hx21-openwrt-kurulumu.4034814/)
- Techolay Sosyal (Turkish) — parallel HX21 install thread, includes a reported bricking incident and community troubleshooting — [techolay.net/sosyal/konu/imou-hx21-openwrt-kurulumu.176033](https://techolay.net/sosyal/konu/imou-hx21-openwrt-kurulumu.176033/)
- GitHub — `afk901/imou-hx21-openwrt` — post-install tweaks/optimization scripts (LED manager, performance tweaks) — [github.com/afk901/imou-hx21-openwrt](https://github.com/afk901/imou-hx21-openwrt)

---

*This guide reflects OpenWrt 25.12.5 as the current stable release and HX21 board support as it exists at time of writing (August 2026). Board support for this device is recent — verify current file names, checksums, and any regressions against the official sources above before relying on this for a production deployment, especially if reading this any significant time after publication.*
