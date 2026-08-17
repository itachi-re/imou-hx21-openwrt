# Imou HX21: Backup, Flash & First-Boot Field Notes

Real commands, real output, real mistakes — reconstructed from three live terminal
sessions flashing OpenWrt onto an Imou HX21. Where the original session hit a wall,
that's kept in as a troubleshooting note rather than smoothed over, since that's
usually the part actually worth reading.

> **Scope note:** this documents the *specific sequence that was actually run* —
> in-place flashing via `mtd write` from a shell already on the stock firmware,
> not the TFTP bootloader-recovery method described elsewhere in this repo's
> `docs/installation-guide.md`. If you don't have SSH access to stock firmware
> the way this session did, use the TFTP method instead; treat this doc as a
> field report on one working path, not the only path.

---

## 0. Starting point: stock firmware already has open root SSH

This is worth calling out on its own, since it wasn't the assumption going in.

```
ssh root@192.168.10.1
BusyBox v1.33.2 (2024-03-29 09:27:14 UTC) built-in shell (ash)
 -----------------------------------------------------
 OpenWrt 21.02-SNAPSHOT, r0-1c660f2d
 -----------------------------------------------------
=== WARNING! =====================================
There is no root password defined on this device!
Use the "passwd" command to set up a new password
in order to prevent unauthorized SSH logins.
--------------------------------------------------
root@HX21-085Z:~#
```

On this unit, stock Imou firmware is itself an OpenWrt 21.02-SNAPSHOT build
(hostname `HX21-085Z`), reachable at **`192.168.10.1`**, with **no root password
set at all** — SSH just drops straight into a root shell. No special unlock
payload or debug-mode trigger was needed to get here on this firmware revision.

**Two things follow from that:**
- If your unit behaves the same way, don't leave it like this even
  temporarily — set a root password (`passwd`) the moment you're in, especially
  before this device touches a WAN connection.
- This may be firmware-revision-specific. Don't assume every HX21 unit ships
  this open; verify on your own device before relying on it.

---

## 1. Back up stock partitions before touching anything

### 1.1 Confirm the MTD layout first

```
root@HX21-085Z:~# cat /proc/mtd
dev:    size   erasesize  name
mtd0: 08000000 00020000 "spi0.0"
mtd1: 00100000 00020000 "BL2"
mtd2: 00080000 00020000 "u-boot-env"
mtd3: 00200000 00020000 "Factory"
mtd4: 00200000 00020000 "FIP"
mtd5: 07280000 00020000 "ubi"
```

Five named partitions plus the raw flash device (`spi0.0`, the whole 128MB chip
seen as one block). `ubi` (`mtd5`) is by far the largest at `0x07280000` ≈ 114.5MB
— that's the actual OS/rootfs partition; everything else is boot chain and
factory config.

### 1.2 Dump the small partitions on-device, then pull them over

```
root@HX21-085Z:~# dd if=/dev/mtd1 of=/tmp/BL2_backup.bin bs=1M count=1
root@HX21-085Z:~# dd if=/dev/mtd2 of=/tmp/u-bootenv_backup.bin bs=512K count=1
root@HX21-085Z:~# dd if=/dev/mtd3 of=/tmp/factory_backup.bin bs=2M count=1
root@HX21-085Z:~# dd if=/dev/mtd4 of=/tmp/FIP_backup.bin bs=2M count=1
```

Block sizes here just need to be ≥ the partition size so `count=1` grabs the
whole thing in one read — `1M` for a 1MB partition, `512K` for 512KB, `2M` for a
2MB partition. Doesn't need to be exact, just not smaller than the partition.

### 1.3 Gotcha: `scp` pull from the router fails outright

```
scp root@192.168.10.1:/tmp/BL2_backup.bin .
ash: /usr/libexec/sftp-server: not found
scp: Connection closed
```

BusyBox's `ash` on this firmware doesn't ship an SFTP subsystem, and modern
`scp` defaults to the SFTP protocol. **Fix: force the legacy SCP protocol with
`-O`:**

```
scp -O root@192.168.10.1:/tmp/BL2_backup.bin .
scp -O root@192.168.10.1:/tmp/u-bootenv_backup.bin .
scp -O root@192.168.10.1:/tmp/factory_backup.bin .
scp -O root@192.168.10.1:/tmp/FIP_backup.bin .
```

```
BL2_backup.bin           100% 1024KB  31.8MB/s   00:00
u-bootenv_backup.bin     100%  512KB  32.0MB/s   00:00
factory_backup.bin       100% 2048KB  29.0MB/s   00:00
FIP_backup.bin           100% 2048KB  29.2MB/s   00:00
```

Clean transfer once `-O` is added. If you hit this on any BusyBox-based
embedded Linux device, not just this router, the fix is the same.

### 1.4 Dumping the 114.5MB `ubi` partition

**The gotcha that ate the most time here: don't nest an SSH `dd` loop inside an
already-open SSH session.** The original attempt ran this *from inside* an
active `ssh root@192.168.10.1` shell:

```
for i in $(seq 0 10 114); do
  echo "Dumping chunk at ${i}MB..."
  ssh root@192.168.10.1 "dd if=/dev/mtd5 bs=1M skip=$i count=10 2>/dev/null" >> ubi_backup.bin
done
```

That spawns a *nested* SSH connection from inside the router's own shell back
to itself, which immediately hits an unrelated host-key prompt
(`Host '192.168.10.1' is not in the trusted hosts file`) and stalls. **Run this
loop from your host machine, in a normal terminal — not from inside a shell
already on the router:**

```sh
# on your host machine, NOT inside an active router SSH session
for i in $(seq 0 10 114); do
  echo "Dumping chunk at ${i}MB..."
  ssh root@192.168.10.1 "dd if=/dev/mtd5 bs=1M skip=$i count=10 2>/dev/null" >> ubi_backup.bin
done
```

```
Dumping chunk at 0MB...
...
Dumping chunk at 110MB...
```

```sh
ls -la ubi_backup.bin
# .rw-r--r-- 120M itachi  2 Aug 10:44  ubi_backup.bin
```

120MB output for a 114.5MB partition is expected — the chunk loop overshoots
slightly by design (`0..114` in steps of 10, last chunk still pulls a full 10MB
even past the partition's real end), and any extra past mtd5's true size will
just be padding/garbage from whatever sits next in flash. Harmless for a backup
whose only purpose is "can I restore from this," but don't assume the file
size tells you the partition size.

### 1.5 Verify the dump integrity

```sh
# on the router (fresh session)
ssh root@192.168.10.1 "dd if=/dev/mtd5 bs=1M count=115 2>/dev/null | md5sum"

# on your host machine
md5sum ubi_backup.bin
```

```
4b51a9b9983ea8896d9b287c57b1b02d  -
4b51a9b9983ea8896d9b287c57b1b02d  ubi_backup.bin
```

Hashes matched — the chunked dump is byte-identical to a direct read of the
same range.

### 1.6 The simpler alternative that also works

Given the hashes matched, the chunking loop wasn't actually necessary here —
this single command produces the same result with far less complexity:

```sh
ssh root@192.168.10.1 "cat /dev/mtd5" > ubi_backup.bin
```

**Recommendation:** try the single-command version first. Fall back to the
chunked loop only if it times out or gets killed partway (more likely on
slower links or if the router's under memory pressure) — chunking gives you
resumability and progress feedback that a single 114MB `cat` doesn't.

---

## 2. Flashing OpenWrt (in-place, via `mtd write` from stock firmware)

### 2.1 Files needed

From an OpenWrt HX21 build (`openwrt-25.12.4-mediatek-filogic-imou_hx21-*`):

| File | Purpose |
|---|---|
| `*-preloader.bin` | First-stage bootloader |
| `*-bl31-uboot.fip` | ARM Trusted Firmware + U-Boot, written to the `FIP` MTD partition |
| `*-initramfs-recovery.itb` | Recovery/rescue image — boots to RAM, doesn't touch flash |
| `*-squashfs-sysupgrade.itb` | The actual OS image for normal sysupgrade-style installs |

### 2.2 Push files to the router

```
scp -O openwrt-25.12.4-mediatek-filogic-imou_hx21-preloader.bin root@192.168.10.1:/tmp/
scp -O openwrt-25.12.4-mediatek-filogic-imou_hx21-bl31-uboot.fip root@192.168.10.1:/tmp/
```

Same `-O` requirement as the backup pulls, since it's the same BusyBox
`scp`/no-SFTP-server situation, just in the other direction this time.

### 2.3 Flash the FIP partition

```
root@HX21-085Z:~# mtd write /tmp/openwrt-mediatek-filogic-imou_hx21-bl31-uboot.fip FIP
Unlocking FIP ...
Writing from /tmp/openwrt-mediatek-filogic-imou_hx21-bl31-uboot.fip to FIP ...
root@HX21-085Z:~# sync
root@HX21-085Z:~# reboot
```

`mtd write <file> <partition-name>` targets the partition by the name shown in
`/proc/mtd`, not a device path — that's what makes this the fast, no-TFTP path
when you already have working root SSH on stock firmware. `sync` before
`reboot` isn't optional here — it flushes any buffered writes to the actual
flash chip before the board power-cycles.

> ⚠️ This overwrites the bootloader/firmware-interface partition on a live,
> running device. If the write is interrupted (power loss, wrong file, wrong
> partition name) you're relying on the TFTP recovery path in
> `docs/installation-guide.md` to get back in. Keep the backups from section 1
> and a TFTP-capable setup on standby before running this.

### 2.4 The router's LAN IP changes after this flash

Stock firmware served SSH at **`192.168.10.1`**. After the FIP flash and
reboot, the board comes back up on OpenWrt's **default `192.168.1.1`** — a
different subnet entirely. If your next `ssh root@192.168.10.1` just hangs or
refuses, that's expected — the device moved, not failed.

---

## 3. Recovering host network connectivity after the reboot

The host's own Ethernet link can get stuck mid-negotiation right after the
router reboots and renegotiates its port. What was seen here, in order:

```
nmcli connection up enp10s0
# Error: Connection activation failed: Disconnected by user
```

```sh
ip a show enp10s0
# link present, UP, but no inet address assigned
```

```sh
nmcli connection modify enp10s0 ipv4.method auto
nmcli connection down enp10s0 && nmcli connection up enp10s0
# Error: Connection activation failed: IP configuration could not be reserved
```

DHCP kept failing because the router hadn't finished bringing its LAN-side
`dnsmasq`/DHCP service back up yet after the reboot. **Working fix: bypass
DHCP and assign a static IP on the same subnet as OpenWrt's default:**

```sh
sudo ip addr flush dev enp10s0
sudo ip addr add 192.168.1.2/24 dev enp10s0
sudo ip link set enp10s0 up
```

```sh
ping 192.168.1.1
# 64 bytes from 192.168.1.1: icmp_seq=1 ttl=64 time=0.065 ms
```

Once the static address was in place, the router answered immediately. If
`ssh root@192.168.1.1` still hangs right after this (it did, briefly, in this
session — `nmap` even reported the host down), give the router 15–30 seconds
past first ping response before retrying SSH; the `dropbear`/SSH daemon
sometimes comes up after ICMP already responds.

---

## 4. First boot on OpenWrt

### 4.1 Confirm you're actually on the new firmware

```
root@OpenWrt:~# cat /etc/openwrt_release
DISTRIB_ID='OpenWrt'
DISTRIB_RELEASE='SNAPSHOT'
DISTRIB_REVISION='r35634-d3191cb056'
DISTRIB_TARGET='mediatek/filogic'
DISTRIB_ARCH='aarch64_cortex-a53'
DISTRIB_DESCRIPTION='OpenWrt SNAPSHOT r35634-d3191cb056'
```

Same "no root password" banner appears again at this point — set one with
`passwd` before doing anything else, same as section 0.

### 4.2 `opkg` doesn't exist — this is an `apk`-based build

```
root@OpenWrt:~# opkg update
-ash: opkg: not found
root@OpenWrt:~# which opkg
root@OpenWrt:~# echo $PATH
/usr/sbin:/usr/bin:/sbin:/bin
```

Confirmed absent, not just missing from `$PATH`. Use `apk` instead — see
`docs/complete-guide.md` for the full command-equivalence table.

```
root@OpenWrt:~# apk
apk-tools 3.0.5, compiled for aarch64.
...
This apk has coffee making abilities.
```

(Yes, that's the real help-text easter egg on this build.)

### 4.3 Getting internet access before WAN is configured

At this point in the session, WAN hadn't been set up yet, so `apk update`
had no route out. This got a temporary path working directly through the
host machine acting as gateway:

```
root@OpenWrt:~# ip route add default via 192.168.1.254
root@OpenWrt:~# echo "nameserver 1.1.1.1" > /etc/resolv.conf
root@OpenWrt:~# ping -c 3 1.1.1.1
64 bytes from 1.1.1.1: seq=0 ttl=55 time=46.814 ms
...
0% packet loss
```

This is a workaround specific to a bench/desk setup where the router's WAN
port isn't plugged into anything yet — not something to leave in place on a
production config. Once WAN is properly configured (DHCP client on the WAN
interface, per `docs/complete-guide.md`), remove this manual route.

### 4.4 Install LuCI

```
root@OpenWrt:~# apk update
[https://downloads.openwrt.org/snapshots/targets/mediatek/filogic/packages/packages.adb]
[https://downloads.openwrt.org/snapshots/packages/aarch64_cortex-a53/base/packages.adb]
...
OK: 11148 distinct packages available

root@OpenWrt:~# apk add luci luci-ssl
( 1/29) Installing cgi-io ...
...
(29/29) Installing luci-ssl ...
```

29 packages pulled in for `luci` + `luci-ssl` on this snapshot build — normal,
LuCI has a fair number of small dependent packages (`liblucihttp`, `ucode`
modules, `rpcd` bindings, per-page LuCI modules).

---

## 5. Wireless configuration

> Values below are placeholders. The original session used SSID and password
> values that shouldn't be reused or published — swap in your own.

```
root@OpenWrt:~# uci set wireless.default_radio0.ssid='<YOUR_SSID_2G>'
root@OpenWrt:~# uci set wireless.default_radio0.encryption='psk2'
root@OpenWrt:~# uci set wireless.default_radio0.key='<YOUR_WIFI_PASSWORD>'
root@OpenWrt:~# uci set wireless.default_radio1.disabled='0'
root@OpenWrt:~# uci set wireless.default_radio1.ssid='<YOUR_SSID_5G>'
root@OpenWrt:~# uci set wireless.default_radio1.encryption='psk2'
root@OpenWrt:~# uci set wireless.default_radio1.key='<YOUR_WIFI_PASSWORD>'
root@OpenWrt:~# uci commit wireless
root@OpenWrt:~# wifi reload
```

`default_radio1` (5GHz) ships **disabled** by default on this board — the
`disabled='0'` line is required, not optional, to bring the 5GHz radio up at
all.

### 5.1 Verify both radios came up

```
root@OpenWrt:~# iwinfo
phy0-ap0 ESSID: "<YOUR_SSID_2G>"
         Mode: Master  Channel: 1 (2.412 GHz)  HT Mode: HE20
         Encryption: WPA-PSK (CCMP)
         Type: nl80211  HW Mode(s): 802.11ax/n/g/b
         Hardware: embedded [MediaTek MT7981]
         PHY name: phy0

phy1-ap0 ESSID: "<YOUR_SSID_5G>"
         Mode: Master  Channel: 36 (5.180 GHz)  HT Mode: HE80
         Center Channel 1: 42
         Encryption: WPA-PSK (CCMP)
         Type: nl80211  HW Mode(s): 802.11ax/ac/n
         Hardware: embedded [MediaTek MT7981]
         PHY name: phy1
```

**Note for the hardware table elsewhere in this repo:** `iwinfo` reports the
radio hardware directly as `embedded [MediaTek MT7981]` on *both* PHYs, not as
a separate companion chip. That's a stronger signal than the earlier
`dmesg`/`iw phy` check (which came up empty for any `mt7976` string) — it
further weakens confidence that a distinct **MT7976C** part is actually on
this board, versus the WiFi being handled by the MT7981 SoC's own integrated
WMAC block for both bands. Worth a footnote or correction on the spec table
either way, since neither piece of evidence collected so far positively
confirms MT7976C.

Channel 36 for 5GHz lands in UNII-1 — a non-DFS channel, so no
radar-detection Channel Availability Check (CAC) delay on startup. If you
change the 5GHz channel later to anything in UNII-2/2e, expect a CAC pause
(typically 60s+) before the radio actually goes live.

### 5.2 Set the regulatory domain and channel width

```
root@OpenWrt:~# uci set wireless.radio1.country='BD'
root@OpenWrt:~# uci commit wireless
root@OpenWrt:~# wifi reload
root@OpenWrt:~# uci set wireless.radio1.htmode='VHT80'
root@OpenWrt:~# uci commit wireless
root@OpenWrt:~# wifi reload
```

Setting `country` isn't cosmetic — it constrains available channels and max
TX power to what's legally permitted in that regulatory domain, and some
channels/widths won't activate at all until a country code is set. Do this
before fine-tuning channel width or TX power.

### 5.3 Confirm the radio actually enabled, not just configured

```
root@OpenWrt:~# logread | grep -i "5g\|mt7976\|DFS\|phy1"
...
daemon.notice hostapd: phy1-ap0: interface state UNINITIALIZED->HT_SCAN
...
daemon.notice hostapd: phy1-ap0: interface state HT_SCAN->ENABLED
daemon.notice hostapd: phy1-ap0: AP-ENABLED
```

`AP-ENABLED` in the hostapd log is the actual confirmation the radio is
broadcasting — `uci commit` + `wifi reload` returning without error only means
the config was accepted, not that the radio came up.

---

## 6. Firewall sanity check

```
root@OpenWrt:~# uci show firewall | grep -i wan
firewall.@zone[1].name='wan'
firewall.@zone[1].network='wan' 'wan6'
firewall.@forwarding[0].dest='wan'
firewall.@rule[0].src='wan'
...
firewall.@rule[8].src='wan'
```

Stock default firewall zone/rule set — a `wan` zone bound to both `wan` and
`wan6` networks, one `forwarding` rule (LAN→WAN), and several `src='wan'`
rules (the default inbound-from-WAN allow/reject rule set OpenWrt ships with,
e.g. ICMP, DHCPv6, IGMP). Worth running this check right after first boot,
before adding any VPN zones from `docs/vpn-netbird.md` or
`docs/vpn-tailscale.md` — confirms you're building on the stock baseline, not
on top of a firewall config that already has cruft in it from a previous
experiment.

---

## 7. Quick troubleshooting reference

| Symptom | Cause | Fix |
|---|---|---|
| `scp: Connection closed` / `sftp-server: not found` | BusyBox `ash` has no SFTP subsystem, modern `scp` defaults to SFTP | Add `-O` to force legacy SCP protocol, both directions |
| Nested `ssh ... dd` loop hangs on a host-key prompt | Running the per-chunk `ssh` loop from *inside* an already-open SSH session to the same router | Run the loop from your host machine's own terminal, not from within a router shell |
| `ubi_backup.bin` is a few MB larger than the partition | `seq 0 10 114` step size overshoots the exact partition boundary slightly | Expected — verify via `md5sum` against a direct read of the same byte range instead of trusting file size |
| `ssh root@192.168.10.1` refuses/hangs after flashing FIP | Router moved to OpenWrt's default `192.168.1.1` after reboot | Point SSH at `192.168.1.1`, not the old stock-firmware IP |
| Host NIC won't get a DHCP lease right after router reboot | Router's LAN-side DHCP service hasn't finished starting yet | Temporarily assign a static IP (`192.168.1.2/24`) on the host instead of waiting on DHCP |
| `opkg: not found` | This build uses `apk`, not `opkg` — don't assume based on OpenWrt version alone | `which opkg` to confirm absence, then use `apk update` / `apk add` |
| `apk update` has no route to the internet | WAN interface not configured yet | Temporary manual default route + `resolv.conf` entry via the host, removed once WAN is set up properly |
| 5GHz radio config committed but nothing broadcasts | `default_radio1.disabled` defaults to `'1'` on this board | Explicitly `uci set wireless.default_radio1.disabled='0'` |
| Unsure if a radio actually came up after `wifi reload` | `uci commit` succeeding only confirms config was accepted | Check `logread \| grep hostapd` for `AP-ENABLED`, not just the absence of a UCI error |

---

## 8. Security notes on this session specifically

- **Rotate any Wi-Fi password that appeared in a terminal log you've saved,
  screenshotted, or uploaded anywhere** — plaintext credentials in scrollback
  are easy to forget about once the terminal's closed.
- **Set a root password immediately** on first SSH access to either the stock
  firmware or fresh OpenWrt — both boot with `=== WARNING! === There is no
  root password defined` and an open shell, which is fine on an isolated bench
  network but not once the device is on a real LAN or has a WAN connection.
- **Rename any SSID that isn't something you'd want broadcasting** — this
  applies whether or not the network is public-facing.
