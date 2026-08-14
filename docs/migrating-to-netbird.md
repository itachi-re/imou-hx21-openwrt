# NetBird on OpenWrt (Imou HX21 / MediaTek MT7981B)

A complete, tested walkthrough for migrating off Tailscale, installing NetBird on an
OpenWrt router, using it as a routing peer for LAN access, and optionally as an
internet exit node.

Written against NetBird's official OpenWrt package feed (docs updated 2026-07-06)
and OpenWrt's `uci`/`opkg`/`apk` tooling. Verify your own OpenWrt release and NetBird
version against the table below before running anything — package managers and
NetBird versions differ across OpenWrt releases.

---

## 0. Check your OpenWrt release first

```
cat /etc/openwrt_release
```

This determines which package manager and which NetBird version you'll get from the
feed:

| OpenWrt release | NetBird version (feed) | Package manager |
| --------------- | ----------------------- | --------------- |
| 23.05           | 0.24.x                  | `opkg`          |
| 24.10           | 0.59.x                  | `opkg`          |
| 25.12 and later | 0.66.x (0.73.x on snapshot) | `apk`       |

> OpenWrt switched its default package manager from `opkg` to `apk` starting with
> 24.x snapshots / stabilizing in 25.x. Use the column above to know which one your
> router has — do not assume.

Verified against a live HX21 running `OpenWrt SNAPSHOT, r35634-d3191cb056`
(`aarch64_cortex-a53`, `mediatek/filogic` target) with `apk`, netbird `0.73.2`.

### Prerequisites

- SSH access to the router (`ssh root@<router-ip>`)
- Enough free flash storage — the NetBird binary is relatively large; low-storage
  routers (small NOR flash devices) may not have room. Check with `df -h`.
- A **setup key** generated from the NetBird dashboard (Team → Setup Keys), or a
  self-hosted NetBird management server URL if you're not using NetBird Cloud.
  In the dashboard: **Create Setup Key** → name it (e.g. `hx21-router`) → pick
  **Reusable** vs **One-off** → set an expiration → optionally auto-assign a group →
  copy the key immediately, it's shown only once.

> The official one-line install script at `pkgs.netbird.io/install.sh` **does not
> support OpenWrt**. Only the OpenWrt package feed method below works.

### On snapshot builds, refresh the apk cache before anything else

If you're on an OpenWrt `SNAPSHOT` build, `apk` will frequently print warnings like:

```
WARNING: opening from cache https://downloads.openwrt.org/snapshots/.../packages.adb: No such file or directory
```

on any package operation until the index cache is repopulated. This is harmless but
means your view of available packages may be stale. Run this once before installing
or removing anything, and confirm it ends clean (no warnings, a distinct-package
count):

```
apk update
```

---

## 1. Migrating from Tailscale? Remove it completely first

Skip this whole section if you're installing NetBird fresh. If you're replacing an
existing Tailscale setup on the same router, do this **before** touching NetBird —
Tailscale and NetBird can technically coexist (different interfaces, `tailscale0` vs
`wt0`), but a clean removal avoids duplicate firewall zones, leftover kernel
modules, and confusing `logread` output later.

### 1.1 Inventory what's installed

```
apk list -I | grep -i tailscale
```

Typical result: the `tailscale` package itself, plus whichever LuCI front-end you
used (`luci-app-tailscale`, `luci-app-tailscale-ng`, or
`luci-app-tailscale-community`).

### 1.2 Log out cleanly before removing anything

Do this first so Tailscale deregisters the node from your tailnet instead of
leaving a stale device entry in the admin console:

```
tailscale down
tailscale logout
```

### 1.3 Stop and disable the service

```
/etc/init.d/tailscale stop
/etc/init.d/tailscale disable
```

You may see this in the output — it's expected and harmless, it's just Tailscale's
own cleanup routine hitting the same missing-binary gap covered in section 3.2
below:

```
linuxfw: clear ip6tables: exec: "ip6tables": executable file not found in $PATH
```

### 1.4 Remove the packages via apk

```
apk del tailscale
```

If a LuCI app is still installed, `apk del tailscale` will refuse with something
like:

```
World updated, but the following packages are not removed due to:
  tailscale: luci-app-tailscale-community
```

Remove the LuCI app(s) first (harmless to run all three even if only one is
installed — the others just no-op):

```
apk del luci-app-tailscale 2>/dev/null
apk del luci-app-tailscale-ng 2>/dev/null
apk del luci-app-tailscale-community 2>/dev/null
```

Then `tailscale` itself will purge cleanly, along with any package-owned orphan
like `kmod-tun` if nothing else on the router needs it:

```
(1/3) Purging luci-app-tailscale-community
(2/3) Purging tailscale
(3/3) Purging kmod-tun
```

### 1.5 Remove leftover state and config files

Package removal cleans up files it owns, but Tailscale intentionally leaves its
node-identity state behind so a reinstall doesn't burn a new key. Since you're
leaving Tailscale for good, wipe it explicitly:

```
rm -rf /etc/tailscale
rm -f /etc/config/tailscaled.state
rm -f /etc/config/tailscale
rm -rf /usr/lib/tailscale
rm -f /usr/bin/tailscale-update
rm -f /var/log/tailscaled.log*
```

If a community LuCI fork was involved, it may keep a *separate* UCI config
alongside the stock one — check and remove it too:

```
ls /etc/config | grep -i tailscale
rm -f /etc/config/luci-app-tailscale-ng
```

### 1.6 Remove the UCI network interface and firewall zone

Anything you configured by hand (subnet routing, a dedicated firewall zone) isn't
touched by `apk del` — find and remove it manually:

```
uci show network | grep -i tailscale
uci show firewall | grep -i tailscale
```

Typical output:

```
network.tailscale=interface
network.tailscale.proto='none'
network.tailscale.device='tailscale0'

firewall.@zone[2].name='tailscale'
firewall.@zone[2].network='tailscale'
firewall.@forwarding[1].dest='tailscale'
firewall.@forwarding[2].src='tailscale'
firewall.@forwarding[3].src='tailscale'
```

```
uci delete network.tailscale
uci commit network
```

> **Firewall zones and forwarding rules added with `uci add` are anonymous
> sections** — referenced by index (`@zone[2]`, `@forwarding[1]`), not by name.
> `uci delete firewall.tailscale_zone` will fail with `Entry not found` even
> though the zone clearly exists — there's no section literally named
> `tailscale_zone`. Delete by index instead, **highest index first**, since
> removing one shifts every index below it:
>
> ```
> uci delete firewall.@forwarding[3]
> uci delete firewall.@forwarding[2]
> uci delete firewall.@forwarding[1]
> uci delete firewall.@zone[2]
> uci commit firewall
> /etc/init.d/firewall restart
> ```
>
> Confirm it's actually gone before moving on:
>
> ```
> uci show firewall | grep -i tailscale
> ```
>
> (should return nothing)

### 1.7 Leave the WireGuard kernel module alone if anything else needs it

Tailscale on OpenWrt normally runs its own userspace WireGuard implementation
rather than depending on `kmod-wireguard`, so there's typically nothing kernel-side
to clean up. If you're also running native OpenWrt WireGuard (client or server)
separately from Tailscale/NetBird, check before removing anything:

```
apk info --rdepends kmod-wireguard
```

If it's required by `wireguard-tools` or anything else you use, leave it — don't
`apk del` it.

### 1.8 Restart networking and firewall, then verify

```
/etc/init.d/network restart
/etc/init.d/firewall restart
```

```
which tailscale tailscaled        # nothing
apk list -I | grep -i tailscale   # nothing
ip link show                      # no tailscale0
uci show network | grep -i tailscale   # nothing
```

A reboot here is a good sanity check before installing NetBird.

---

## 2. Install the NetBird package

### OpenWrt 24.10 and earlier (`opkg`)

```
opkg update
opkg install netbird
```

### OpenWrt 25.12 and later (`apk`)

```
apk update
apk add netbird
```

Both install a `procd` init script at `/etc/init.d/netbird`.

---

## 3. Enable and start the service

```
/etc/init.d/netbird enable
/etc/init.d/netbird start
```

At this point the daemon is running but not yet authenticated to any NetBird
network — it's waiting for a login.

### 3.1 First-run sanity check

```
netbird version
/etc/init.d/netbird status
```

Should report a version string and `running`.

### 3.2 Known gap on `firewall4`/nftables builds: missing `ip6tables`

If `netbird status -d` (or the daemon log via `logread | grep -i netbird`) errors
with:

```
Error: status failed: create firewall manager: create firewall: create IPv6 firewall: init ip6tables: exec: "ip6tables": executable file not found in $PATH
```

...and `ip link show wt0` reports `Device "wt0" does not exist` — this is because
modern OpenWrt (`firewall4`/nftables) ships `nft` by default but **not** the legacy
`iptables`/`ip6tables` command shims that NetBird's Go firewall manager shells out
to directly. This is the same class of issue Tailscale hits on the same firewall
stack (see the note in section 1.3 above).

**This bit us specifically on first boot after a reboot** — the daemon started,
`enabled` returned `0`, `ps` showed the process running, but `wt0` never got
created until this was fixed:

```
apk update
apk add iptables-nft ip6tables-nft
/etc/init.d/netbird restart
```

Then confirm:

```
which ip6tables          # /usr/sbin/ip6tables
ip link show wt0         # should now exist, UP
netbird status -d        # should no longer error
```

These packages persist across reboots the same way `netbird` itself does (both
live on the writable overlay), so this is a one-time fix, not something to repeat
every boot.

---

## 4. Connect the router to your NetBird network

### NetBird Cloud, with a setup key

```
netbird up --setup-key <SETUP_KEY>
```

### Self-hosted NetBird

```
netbird up --setup-key <SETUP_KEY> --management-url https://netbird.example.com:443
```

Replace the URL with your own management server's address.

### Verify

```
netbird status -d
```

`-d` gives the detailed/debug view — confirm `Management: Connected` and `Signal: Connected`, and that a peer IP has been assigned.

Once connected, the router shows up on the **Peers** page of your NetBird dashboard.
NetBird creates and fully manages its own WireGuard interface named **`wt0`**.

> **Do not** touch `wt0` through OpenWrt's LuCI WireGuard UI or manual `uci` WireGuard
> config. NetBird owns that interface and its keys entirely — manual edits will
> conflict with it and can break the connection.

### 4.1 A peer showing `Status: Idle` / `Peers count: 0/1 Connected` isn't necessarily broken

If `netbird status -d` shows a peer with `Status: Idle`, empty `Connection type`,
and no WireGuard handshake, don't assume something's misconfigured — check
`Lazy connection: true` in the same output first. With lazy connections enabled
(the default), NetBird doesn't establish a WireGuard session to a peer until
traffic actually needs to flow to it. `Idle` at rest is the expected steady state.

To confirm the tunnel actually works, generate real traffic and re-check:

```
ping -c 4 <peer's NetBird IP>
netbird status -d
```

A working tunnel flips to `Status: Connected`, populates `Connection type` (`P2P`
or `Relayed`), and shows a recent `Last WireGuard handshake`. `Connection type:
Relayed` (via a `rels://...relay.netbird.io` address) is normal and fully
functional — it just means direct P2P hole-punching didn't succeed (common behind
CGNAT or restrictive NAT on either peer), so NetBird transparently falls back to
its relay infrastructure. Expect somewhat higher latency on a relayed path than a
direct one; if you want to chase direct P2P instead, that usually means forwarding
NetBird's WireGuard port (`51820/udp`) on whatever sits upstream of the router's
WAN — not required for correct operation, just for the latency win.

---

## 5. DNS configuration (optional, only if you use NetBird DNS features)

OpenWrt's `dnsmasq` already listens on port 53, which conflicts with NetBird's own
managed DNS resolver. If you want NetBird DNS features (domain resources, peer name
resolution via MagicDNS-equivalent), you need to run NetBird's resolver on an
alternate port and forward the relevant domains to it from dnsmasq.

If you don't need domain resources / peer-name resolution, **skip this section
entirely** — plain connectivity works without it.

### 5.1 Pin NetBird's DNS resolver to an alternate port

```
netbird up --dns-resolver-address 127.0.0.1:5053
```

If already connected, this returns `Already connected` and simply updates the
running config — no need to `down`/`up` around it.

(NetBird auto-falls-back to another port if 53 is taken anyway, but pinning it keeps
the dnsmasq forwarding rule below valid and predictable.)

### 5.2 Forward NetBird's DNS domain to that port via dnsmasq

Replace `netbird.cloud` with `netbird.selfhosted` (or your custom DNS domain) if
you're self-hosting:

```
uci add_list dhcp.@dnsmasq[0].server='/netbird.cloud/127.0.0.1#5053'
uci commit dhcp
/etc/init.d/dnsmasq restart
```

> `/etc/init.d/dnsmasq restart` can transiently disrupt DHCP on the LAN — expect a
> possible `udhcpc: no lease, failing` line from any client mid-renewal at that
> exact moment. It self-heals on the next DHCP renewal/reboot; don't chase it as a
> separate bug.

---

## 6. Use the router as a routing peer (LAN access from other NetBird peers)

By default, the router is just a normal client peer — nothing else on your LAN can
be reached through it, and it doesn't route anyone else's traffic. To let other
NetBird peers reach devices on the router's LAN (or vice versa), configure it as a **routing peer**.

### 6.1 Register the `wt0` interface in OpenWrt's network config

```
uci set network.netbird=interface
uci set network.netbird.proto='none'
uci set network.netbird.device='wt0'
uci commit network
```

### 6.2 Create a firewall zone and allow forwarding

```
uci add firewall zone
uci set firewall.@zone[-1].name='netbird'
uci set firewall.@zone[-1].input='ACCEPT'
uci set firewall.@zone[-1].output='ACCEPT'
uci set firewall.@zone[-1].forward='ACCEPT'
uci set firewall.@zone[-1].masq='1'
uci add_list firewall.@zone[-1].network='netbird'

uci add firewall forwarding
uci set firewall.@forwarding[-1].src='lan'
uci set firewall.@forwarding[-1].dest='netbird'

uci add firewall forwarding
uci set firewall.@forwarding[-1].src='netbird'
uci set firewall.@forwarding[-1].dest='lan'

uci commit firewall
/etc/init.d/firewall restart
```

This permits all traffic on the NetBird interface in both directions. Access is
still governed by **NetBird's own ACL/access-control policies** in the dashboard —
this firewall config just stops OpenWrt itself from blocking the interface.

> **Before running this**, check `uci show firewall` for any zone that the NetBird
> package may have already auto-created (some package builds add one on install,
> similar to how the Tailscale OpenWrt package does). If a `netbird` zone already
> exists, don't add a duplicate — you'll hit the same `redefinition of symbol` nftables error you'd get from duplicating any zone name.
> Check first with:
>
> ```
> uci show firewall | grep -i netbird
> ```

### 6.3 Advertise your LAN as a network resource

In the NetBird dashboard: **Networks → Add Network Resource**, add the router's LAN
subnet (e.g. `192.168.1.0/24`), and set this router as the routing peer. Then create
an access-control policy granting the groups you want access to that resource.

---

## 7. Use the router as an internet exit node (route your traffic through the router's WAN IP)

This is the "make my router's IP browse the internet" use case — all traffic from
your other devices (phone, laptop) egresses through the router's WAN connection and
appears with the router's public IP.

Requires **NetBird v0.27.0+** (irrelevant here since the router only needs to *be* the exit node — the version requirement is what enforces default-route + masquerade
support server-side).

### 7.1 In the NetBird dashboard

1. Go to **Peers**, find the router (it should already be listed and connected from
   section 4).
2. Click **Add Exit Node** on that peer.
3. Assign one or more **distribution groups** — these determine which peers (e.g.
   your phone) are allowed to route through this exit node.
4. Decide on **Auto Apply**:
   - Enabled (default on clients v0.55.0+): peers in the distribution group start
     using the exit node automatically once it's available.
   - Disabled: peers can still select it manually from their own NetBird client.
5. Confirm — masquerade is on by default and required for exit-node mode
   (it rewrites outbound packets to appear as if they came from the router's WAN IP).
6. NetBird auto-creates the `0.0.0.0/0` route, and a matching `::/0` route if IPv6
   overlay addressing is enabled on the peer. Peers without IPv6 enabled have their
   IPv6 traffic blocked outright rather than leaking outside the tunnel.

### 7.2 On your phone

Open the NetBird app → select the router as the exit node (or confirm it applied
automatically if Auto Apply was left on).

### 7.3 Verify

From the phone, check `https://ifconfig.me` (or similar) — it should now show the
router's WAN IP.

### 7.4 Known caveat — LAN exposure through exit nodes

There's an open upstream issue (netbirdio/netbird #5797) where devices using a
router as an exit node can reach the **entire LAN subnet** of that router — not just
the internet — even with a custom ACL policy and no explicit LAN route advertised.
This happens because the exit node's masquerade rule forwards everything arriving on `wt0` to the WAN interface *and* the LAN forwarding path is not restricted at the
kernel level by NetBird's peer-level ACLs.

**Mitigation**, per the NetBird docs' own guidance: assign an access-control group
to the `0.0.0.0/0` route even though it's optional. An empty/unmatched
access-control group with no policy referencing it causes NetBird to drop routed
(non-internet) traffic by default — internet traffic itself is unaffected since it's
handled by masquerade before route ACLs are evaluated. Until this is resolved
upstream, don't assume LAN isolation is automatic — test it yourself by pinging the
router's LAN IP (e.g. `192.168.1.1`) from the phone while connected through the exit
node, and confirm it's actually blocked.

---

## 8. Persistence across reboots and firmware upgrades

The OpenWrt package marks NetBird's config as a conffile, so it survives `sysupgrade` without extra steps. Storage location depends on your package version:

- OpenWrt 25.12+: `/root/.config/netbird`
- OpenWrt 24.10 and earlier: `/etc/netbird/config.json`

Both paths are included automatically in the `sysupgrade` config backup.

Confirmed by reboot test on the HX21: `/root/.config/netbird/` (containing
`active_profile.json`, `active_profile.txt`, `default.json`, and a `root/` peer-key
subdirectory) survived a reboot intact — the daemon reused the existing identity
without re-registering. The `wt0` interface simply failed to come up until the
`ip6tables-nft` gap (section 3.2) was fixed; the connection state itself was never
lost.

---

## 9. Quick troubleshooting reference

| Symptom                                                       | Likely cause                                                              | Check                                                                                    |
| --------------------------------------------------------------- | ---------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------ |
| `apk del tailscale` refuses, lists a `luci-app-tailscale*` dependent | LuCI front-end still installed                                          | `apk del` the LuCI app first, then retry `apk del tailscale`                              |
| `uci delete firewall.<name>_zone` fails with `Entry not found`  | Zone/forwarding was added anonymously via `uci add`, has no literal name    | Find its real index with `uci show firewall`, delete by `@zone[N]` / `@forwarding[N]`, highest index first |
| `netbird status -d` errors: `ip6tables: executable file not found in $PATH`, `wt0` doesn't exist | Missing `iptables-nft`/`ip6tables-nft` compat shims on a `firewall4`/nftables build | `apk add iptables-nft ip6tables-nft`, then `/etc/init.d/netbird restart`                  |
| `netbird up` hangs or fails to authenticate                     | Setup key expired/already used, or wrong `--management-url`                  | Generate a fresh one-time or reusable key in the dashboard; confirm URL if self-hosted     |
| Peer shows `Idle`, `0/N Connected`, no handshake                | Normal with `Lazy connection: true` and no active traffic yet — not a bug by itself | Generate traffic (`ping <peer NetBird IP>`) and re-check `netbird status -d`               |
| Peer shows in dashboard but no traffic flows                    | Firewall zone/forwarding not applied, or `wt0` not in a zone at all          | `uci show firewall | grep -i netbird`, `ip a show wt0`                                    |
| Duplicate zone / `redefinition of symbol` on firewall restart   | Package auto-created a `netbird` zone and you added a second one manually    | `uci show firewall`, delete the duplicate `cfgXXXXXX` entry, not the package-managed one   |
| DNS-based domain resources not resolving                        | Port 53 conflict with dnsmasq never resolved                                 | Complete section 5, confirm with `netbird status -d` that a DNS resolver line is listed    |
| `udhcpc: no lease, failing` right after `dnsmasq restart`       | Transient DHCP disruption from the restart itself                            | Harmless, self-heals on next renewal/reboot — not related to NetBird                       |
| Exit node traffic reaches router's LAN devices                  | Known upstream issue #5797                                                   | Apply the access-control-group mitigation in section 7.4 and test manually                 |
| Config lost after firmware upgrade                              | Wrong path assumption for your OpenWrt version                               | Confirm which path in section 8 matches your release before upgrading                      |

---

## Notes on why this differs from Tailscale on the same router

- NetBird creates `wt0`; Tailscale creates `tailscale0` — don't mix firewall rules
  between the two if you're running both for comparison; they need separate zones.
- Both hit the exact same `ip6tables` binary-not-found gap on `firewall4`/nftables
  builds (see section 3.2) — it's a platform gap, not something specific to either
  VPN daemon. If you migrate between them, install `iptables-nft`/`ip6tables-nft`
  once and both benefit.
- NetBird's exit-node ACL model is enforced primarily through **routes + access
  control groups** in the dashboard, not through the OpenWrt firewall itself — the
  local `uci` firewall rules here exist only to stop OpenWrt from blocking the
  tunnel interface, not to enforce NetBird-level policy.
- Unlike Tailscale's OpenWrt package (which is community/upstream OpenWrt-maintained
  and has had longer field exposure on embedded targets), NetBird's own docs note
  x86_64 as the most tested platform, with arm64/other architectures "supported but
  less tested." The Imou HX21 (MT7981B, `aarch64_cortex-a53`) falls in that
  less-tested bucket — keep an eye on `dmesg` and `logread` after installation for
  anything unusual. In practice, on this router the only real-world gap found was
  the `ip6tables-nft` dependency, not anything MT7981B/arm64-specific.

---

## Sources

- NetBird OpenWrt Installation docs (docs.netbird.io/get-started/install/openwrt)
- NetBird Exit Node configuration docs (docs.netbird.io/manage/network-routes/use-cases/exit-nodes)
- NetBird "How Routing Peers Work" docs (docs.netbird.io/manage/networks/how-routing-peers-work)
- netbirdio/netbird GitHub issue #5797 (LAN subnet leak through exit nodes)
- Live migration log on this HX21 (Tailscale removal → NetBird install →
  `ip6tables-nft` fix → routing-peer setup → verified relayed connection),
  2026-08-14
