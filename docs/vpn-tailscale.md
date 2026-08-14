# Tailscale on OpenWrt (Imou HX21 / MediaTek MT7981B)

A complete, researched walkthrough for installing Tailscale on an OpenWrt router,
using it as a subnet router for LAN access, as an internet exit node, and for
Tailscale SSH — plus the firewall/DNS gotchas specific to OpenWrt's
`firewall4`/nftables stack.

Written against the official OpenWrt package feed for Tailscale and Tailscale's own
CLI reference docs. Verify your own OpenWrt release and Tailscale version against
the table below before running anything — package managers and available versions
differ across OpenWrt releases, and instructions found online for `opkg` will not
work verbatim on an `apk`-based router or vice versa.

---

## 0. Check your OpenWrt release and package manager first

```
cat /etc/openwrt_release
```

| OpenWrt release | Package manager | Notes |
| ---------------- | ---------------- | ------ |
| 23.05             | `opkg`            | Minimum release for the official `tailscale` feed package |
| 24.10             | `opkg`            | |
| 25.12 and later   | `apk`             | OpenWrt switched its default package manager from `opkg` to `apk` starting with 24.x snapshots, stabilizing in 25.x |

> The official `tailscale` opkg/apk package and `luci-app-tailscale` require
> **OpenWrt ≥ 23.05** and enough free flash (Tailscale needs roughly 20–25 MB).
> Very low-flash devices (a few MB of usable space) may not have room for the
> official package — third-party UPX-compressed builds exist for that case, but
> this guide covers the official feed package only, which is the supported path.

Do not assume which package manager you have — check first. Instructions for one
will silently fail or behave unexpectedly on the other.

### Prerequisites

- SSH access to the router (`ssh root@<router-ip>`)
- Enough free flash storage — check with `df -h`
- A Tailscale account (free tier: up to 3 users / 100 devices, 1 subnet router at
  time of writing — check current limits in your account if this matters to you)
- Either:
  - Nothing extra, if you'll authenticate interactively via a browser URL, **or**
  - A **pre-auth key** generated from the admin console (Settings → Keys → Generate
    auth key) if you want a non-interactive, scriptable login — useful for
    unattended installs or reinstall-after-reset scenarios

---

## 1. Install the package

### OpenWrt 24.10 and earlier (`opkg`)

```
opkg update
opkg install tailscale
```

### OpenWrt 25.12 and later (`apk`)

```
apk update
apk add tailscale
```

Both pull the official package from the OpenWrt feed and install a `procd` init
script at `/etc/init.d/tailscale`. The package also pulls in `kmod-tun` as a
dependency — Tailscale on OpenWrt uses the kernel TUN device by default (not
userspace networking mode), which is what makes the `tailscale0` interface a real
kernel network interface rather than a SOCKS5/HTTP proxy.

> **On snapshot builds**, `apk update` may print
> `WARNING: opening from cache .../packages.adb: No such file or directory`
> the first time you run it — this just means the local index cache is stale/empty
> and is refreshing. Harmless, but make sure the command ends clean (a distinct
> package count, no lingering errors) before proceeding.

### Optional: LuCI web UI

Three community LuCI front-ends exist if you'd rather manage Tailscale from the
web UI instead of SSH:

- `luci-app-tailscale` (the original, from asvow) — takes ownership of the stock
  `/etc/init.d/tailscale` and `/etc/config/tailscale` files; uninstalling it also
  removes those files, breaking the stock package until you reinstall it.
- `luci-app-tailscale-ng` — non-invasive alternative; keeps its own separate UCI
  config (`/etc/config/luci-app-tailscale-ng`) instead of overwriting the stock
  files, so it can be removed without breaking the underlying Tailscale install.
- `luci-app-tailscale-community` — a third variant maintained under the official
  `openwrt/luci` applications tree.

Install whichever you prefer with `apk add luci-app-tailscale-community` (or the
equivalent package name) — none are required; everything in this guide works from
the CLI alone.

---

## 2. Enable and start the service

```
/etc/init.d/tailscale enable
/etc/init.d/tailscale start
```

At this point the daemon is running but not yet authenticated — it's waiting for
`tailscale up`.

Sanity-check:

```
tailscale version
/etc/init.d/tailscale status
```

Should report a version string and a running state.

---

## 3. Known gap on `firewall4`/nftables builds: missing `ip6tables`

Before authenticating, be aware of this — it's easy to hit and the error message
doesn't make the cause obvious.

Modern OpenWrt (`firewall4`, based on `nftables`, default since 22.03) ships `nft`
but **not** the legacy `iptables`/`ip6tables` command shims. Tailscale's firewall
manager (`linuxfw`) shells out to those commands directly to manage its own
NAT/forwarding rules when in its default `nftables` compatibility mode. Without
them present, you'll see errors like this in `logread` or on `tailscale down`:

```
linuxfw: clear ip6tables: exec: "ip6tables": executable file not found in $PATH
```

or, more disruptively, `tailscaled` failing to fully initialize its firewall rules
on `tailscale up`, leaving `tailscale0` half-configured.

**Fix** — install the nft-backed compatibility shims before authenticating:

```
apk update
apk add iptables-nft ip6tables-nft
```

(`opkg install iptables-nft ip6tables-nft` on 24.10 and earlier.)

Then restart the service:

```
/etc/init.d/tailscale restart
```

This is the exact same class of issue NetBird hits on the same firewall stack —
it's a platform gap, not something specific to Tailscale. If you've already fixed
it for another VPN daemon on this router, you're covered here too.

---

## 4. Authenticate the router

### Interactive (browser-based) login

```
tailscale up
```

This prints a login URL — open it in a browser, sign in, and approve the device.
Once approved, `tailscale status` on the router should show it as connected.

### Non-interactive, with a pre-auth key

```
tailscale up --authkey=<AUTH_KEY>
```

Useful for scripted installs. Reusable keys can register multiple devices;
one-off keys are consumed on first use. Both can be set to expire.

### Verify

```
tailscale status
tailscale ip -4
```

`tailscale status` lists peers and connection state; `tailscale ip -4` shows the
`100.x.y.z` address assigned to this router. Once connected, the device also shows
up on the **Machines** page of the Tailscale admin console.

> Tailscale creates and manages its own interface, `tailscale0`, entirely on its
> own. As with any Tailscale/NetBird-style daemon, don't try to configure
> `tailscale0` through OpenWrt's LuCI WireGuard page or hand-edit its `uci` network
> settings the way you might a normal WireGuard interface — the daemon owns the
> interface, its keys, and its routing table entries directly.

---

## 5. Firewall integration

Unlike some other OpenWrt VPN packages, the official Tailscale package **does not**
automatically create an OpenWrt firewall zone for `tailscale0` — its own `linuxfw`
firewall manager handles NAT/forwarding for the tailnet traffic itself at the
nftables/iptables level, separately from OpenWrt's `firewall4`/UCI configuration.
For plain client connectivity (the router reaching your tailnet, nothing routing
through it), **no manual firewall zone is required** — skip straight to section 6
or 7 if that's all you need.

If you want other **LAN devices** behind the router to be reachable over Tailscale
(i.e. you're setting this up as a subnet router — see section 7), or you want
finer per-service control over what's reachable through `tailscale0` from OpenWrt's
own firewall (independent of Tailscale's own filtering), create a dedicated zone:

```
uci add firewall zone
uci set firewall.@zone[-1].name='tailscale'
uci set firewall.@zone[-1].input='ACCEPT'
uci set firewall.@zone[-1].output='ACCEPT'
uci set firewall.@zone[-1].forward='ACCEPT'
uci add_list firewall.@zone[-1].network='tailscale'

uci add firewall forwarding
uci set firewall.@forwarding[-1].src='lan'
uci set firewall.@forwarding[-1].dest='tailscale'

uci add firewall forwarding
uci set firewall.@forwarding[-1].src='tailscale'
uci set firewall.@forwarding[-1].dest='lan'

uci commit firewall
/etc/init.d/firewall restart
```

This also requires registering the interface in the network config, same pattern
as any unmanaged interface:

```
uci set network.tailscale=interface
uci set network.tailscale.proto='none'
uci set network.tailscale.device='tailscale0'
uci commit network
```

> **Check for an existing zone first.** Depending on install method and package
> version, a zone may already exist. Adding a duplicate zone with the same name
> produces an nftables `redefinition of symbol` error on firewall restart. Check
> before adding:
>
> ```
> uci show firewall | grep -i tailscale
> ```

> **Anonymous UCI sections are referenced by index, not name.** If you later need
> to remove these, `uci delete firewall.tailscale_zone` will fail with
> `Entry not found` — there's no section literally named that. Find the actual
> index first (`uci show firewall | grep -i tailscale`, e.g. `@zone[2]`,
> `@forwarding[1]`), then delete by index, **highest index first** since removing
> one shifts the ones below it:
>
> ```
> uci delete firewall.@forwarding[N]
> uci delete firewall.@zone[N]
> uci commit firewall
> /etc/init.d/firewall restart
> ```

### On `--netfilter-mode`

Tailscale's `tailscale up` accepts a `--netfilter-mode` flag (`on` / `nodivert` /
`off`) that controls how aggressively `tailscaled` manages its own iptables/nft
rules. The default (`on`) is fine on OpenWrt once `iptables-nft`/`ip6tables-nft`
are present (section 3). If you want OpenWrt's own `firewall4`/UCI config to be the
single source of truth for all forwarding decisions instead of letting Tailscale
manage its own rules underneath it, you can disable Tailscale's self-management:

```
tailscale up --netfilter-mode=off
```

This is optional and mostly relevant if you're debugging conflicting rules between
Tailscale and a heavily customized OpenWrt firewall config — most setups don't need
it.

---

## 6. DNS and MagicDNS

Tailscale's DNS works differently from most VPN daemons: it doesn't try to bind
your router's local port 53. Instead, Tailscale runs a DNS proxy reachable at the
fixed address **`100.100.100.100`**, accessible only over the `tailscale0`
interface (sometimes called "quad100"). There's no port-53 conflict with
`dnsmasq` on the router itself the way there is with daemons that try to run a
local resolver — but there are still two separate decisions to make.

### 6.1 Should the router itself accept Tailscale's DNS settings?

By default, Linux clients accept DNS config pushed from the admin console
(MagicDNS, split-DNS, etc.) and rewrite `/etc/resolv.conf`. On an OpenWrt router,
`dnsmasq` already owns DNS resolution for the whole LAN — letting Tailscale rewrite
`resolv.conf` on the router itself can fight with dnsmasq's own configuration.
Unless you specifically want the router to resolve `*.ts.net` MagicDNS names
itself, disable this:

```
tailscale up --accept-dns=false
```

(Combine with whichever other flags you're using, e.g.
`tailscale up --accept-dns=false --advertise-routes=192.168.1.0/24`.)

### 6.2 Let LAN clients resolve MagicDNS names via dnsmasq

If you want devices on your LAN (not just the router) to resolve tailnet peer
names (`*.ts.net`) through your existing local DNS, forward that domain to
Tailscale's quad100 resolver from dnsmasq:

```
uci add_list dhcp.@dnsmasq[0].server='/ts.net/100.100.100.100'
uci commit dhcp
/etc/init.d/dnsmasq restart
```

> Restarting dnsmasq briefly interrupts DHCP on the LAN — a client mid-renewal at
> that exact moment may log a transient `udhcpc: no lease, failing`. This is
> harmless and self-heals on the next renewal or reboot.

This only makes sense if the router is reachable from your LAN clients' DNS path
(the normal case) — and note it only forwards the `ts.net` domain itself; it
doesn't require or imply `--accept-dns` on the router.

---

## 7. Subnet router (advertise your LAN to the tailnet)

This lets other devices on your tailnet reach machines on the router's LAN subnet
— not just the router itself.

### 7.1 Confirm IP forwarding is enabled

OpenWrt routers almost always already have this on by default (it's required for
normal LAN↔WAN routing), but confirm rather than assume:

```
sysctl net.ipv4.ip_forward
sysctl net.ipv6.conf.all.forwarding
```

Both should read `= 1`. If either is `0` (unusual on a router, but possible on a
dumb-AP or bridge-mode config), enable it:

```
echo 'net.ipv4.ip_forward=1' >> /etc/sysctl.d/99-tailscale.conf
echo 'net.ipv6.conf.all.forwarding=1' >> /etc/sysctl.d/99-tailscale.conf
sysctl -p /etc/sysctl.d/99-tailscale.conf
```

### 7.2 Advertise the route

```
tailscale up --advertise-routes=192.168.1.0/24 --accept-dns=false
```

Adjust the subnet to match your actual LAN (`uci show network.lan` if unsure).
Comma-separate multiple subnets if needed:
`--advertise-routes=192.168.1.0/24,192.168.2.0/24`.

By default this also NATs (masquerades) subnet traffic so it appears to come from
the router rather than the original tailnet peer's IP
(`--snat-subnet-routes=true` is the default; set `=false` only if you specifically
need the original source IP preserved on your LAN, and be aware Tailscale's own
docs note combining `--snat-subnet-routes=false` with exit-node mode on the same
device can cause upstream traffic drops — keep those two roles on separate devices
if you need both).

### 7.3 Approve the route in the admin console

Advertising a route is not enough on its own — it requires explicit approval:

- Go to the **Machines** page in the Tailscale admin console
- Find the router, open its menu → **Edit route settings**
- Enable the advertised subnet
- Save

Until approved, the route exists but traffic won't actually flow through it.

### 7.4 Have other tailnet devices accept the route

Route advertisement is opt-in on the receiving side too. On another device:

```
tailscale up --accept-routes
```

(Some platforms — Windows, iOS, Android, macOS App Store/standalone — accept
routes by default; Linux and most others default to not accepting them, so this is
usually required explicitly.)

### 7.5 Verify

From another tailnet device:

```
ping 192.168.1.1   # or any LAN device's address
```

If it doesn't respond, re-check: route approved in the admin console, subnet
correctly advertised, `--accept-routes` set on the client, and (if you built the
zone in section 5) the OpenWrt firewall isn't blocking forwarding.

---

## 8. Exit node (route all internet traffic through the router)

This makes your other devices' internet traffic appear to originate from this
router's WAN IP — useful for using trusted home internet from an untrusted
network, or a consistent public IP for geographic reasons.

### 8.1 Advertise the router as an exit node

```
tailscale up --advertise-exit-node
```

### 8.2 Approve it in the admin console

Same as subnet routes — advertising isn't enough on its own:

- **Machines** page → find the router → it should show an **Exit Node** badge
  once advertised
- Open its menu → **Edit route settings** → check **Use as exit node** → Save

### 8.3 Permission model

By default, any user in your tailnet can use any configured exit node — no
explicit ACL grant is needed unless you've already replaced the default "allow
all" policy with a custom one. If you have, you need an explicit grant with
`"dst": ["autogroup:internet"]` for whichever users/groups should be allowed to
route through exit nodes at all (see section 10).

### 8.4 Use it from a client device

```
tailscale up --exit-node=<router's tailnet name or IP>
```

To also keep direct access to your home LAN while routing everything else through
the exit node:

```
tailscale up --exit-node=<router> --exit-node-allow-lan-access
```

On mobile apps, this is a toggle in the Exit Node section of the app instead of a
CLI flag.

### 8.5 Verify

From the client:

```
tailscale status   # should show "exit node: <router-name>"
curl https://ifconfig.me
```

The reported IP should match the router's WAN address.

### 8.6 Caveat worth knowing about

Running a device as **both** a subnet router and an exit node with
`--snat-subnet-routes=false` can cause upstream traffic drops per Tailscale's own
documentation — if you need both roles with SNAT disabled, split them across two
devices. This isn't specific to OpenWrt, but worth knowing if you're combining
sections 7 and 8 on the same HX21.

---

## 9. Tailscale SSH (optional)

Tailscale can act as an SSH server itself, authenticated entirely through your
tailnet identity instead of separately managed SSH keys:

```
tailscale up --ssh
```

Access is governed by your tailnet's ACL policy (or the default "allow all within
tailnet" policy if you haven't customized one). Before relying on this as your
only way to reach the router, confirm your ACL policy actually permits it for your
account — locking yourself out of both regular SSH and Tailscale SSH on a remote
router is a real risk worth testing carefully, ideally with local/console access
as a fallback while you confirm it works.

---

## 10. Access control (ACLs) — brief overview

By default, a fresh tailnet uses a permissive "allow all" policy — every device can
reach every other device. For anything beyond a small personal setup, you'll
likely want to define access controls under **Access Controls** in the admin
console, written as a JSON policy (`grants`/`acls`, `groups`, `tagOwners`, etc.).
This is Tailscale-side configuration, not anything specific to OpenWrt — the
router just needs `tailscale up --ssh` / `--advertise-routes` / etc. to have
already been run for those capabilities to exist as something an ACL policy can
grant or restrict. Covering ACL syntax in full is out of scope here; see
Tailscale's own access control docs if you need it.

---

## 11. Persistence across reboots and firmware upgrades

The package marks Tailscale's configuration as `conffiles`, so it survives
`sysupgrade` without extra steps:

- **Settings/UCI config**: `/etc/config/tailscale`
- **Node identity/state**: `/etc/config/tailscaled.state` — this is what lets the
  router reconnect after a reboot without re-registering as a new device. Treat it
  as sensitive; anyone with this file can potentially impersonate the node.
- **Additional local state**: `/etc/tailscale/`

All of these are included automatically in the `sysupgrade` config backup.

---

## 12. Removing Tailscale

If you ever need to migrate off Tailscale entirely (to NetBird or otherwise), see
the companion doc in this repo — `docs/vpn-netbird.md` — section 1, which covers
full removal: logging out cleanly, purging the package and any LuCI front-end,
wiping leftover state files, and cleaning up UCI network/firewall sections by
index (the same anonymous-section gotcha noted in section 5 above applies there
too).

---

## 13. Quick troubleshooting reference

| Symptom | Likely cause | Check |
| -------- | ------------- | ------ |
| `linuxfw: clear ip6tables: exec: "ip6tables": executable file not found in $PATH` | Missing `iptables-nft`/`ip6tables-nft` on a `firewall4`/nftables build | `apk add iptables-nft ip6tables-nft` (or `opkg install` on 24.10-), then `/etc/init.d/tailscale restart` |
| `tailscale up` hangs or the login URL never gets approved | Auth key expired/already consumed, or you're waiting on manual approval | Generate a fresh key, or check the admin console for a pending device needing manual approval |
| `uci delete firewall.<name>_zone` fails with `Entry not found` | Zone/forwarding rule was added anonymously via `uci add`, has no literal name | Find its real index with `uci show firewall`, delete by `@zone[N]`/`@forwarding[N]`, highest index first |
| Duplicate zone / `redefinition of symbol` on firewall restart | A `tailscale` zone already existed and a second one got added manually | `uci show firewall | grep -i tailscale`, delete the duplicate `cfgXXXXXX` entry, keep the original |
| Subnet route advertised but nothing reaches the LAN | Route not yet approved in the admin console, or `--accept-routes` missing on the client | Check **Machines → Edit route settings** on the router; check `tailscale status` on the client for `--accept-routes` |
| Exit node selected but client has no internet | `--advertise-exit-node` not approved in admin console, or SNAT/masquerade conflict with a subnet-router role on the same device | Confirm the Exit Node badge and approval on the **Machines** page; if also a subnet router, check section 8.6 |
| MagicDNS names (`*.ts.net`) don't resolve for LAN clients | dnsmasq never configured to forward the `ts.net` domain to `100.100.100.100` | Complete section 6.2, confirm with `nslookup <peer>.ts.net` from a LAN client |
| Router's own DNS behaving oddly after `tailscale up` | `--accept-dns` defaulted to true and Tailscale rewrote `resolv.conf`, fighting dnsmasq | Re-run with `--accept-dns=false` (section 6.1) |
| `udhcpc: no lease, failing` right after `dnsmasq restart` | Transient DHCP disruption from the restart itself | Harmless, self-heals on next renewal/reboot — unrelated to Tailscale |
| Locked out of SSH after enabling `--ssh` | ACL policy doesn't actually grant your account SSH access | Use local/console access to fix the ACL policy or re-run without `--ssh` |
| Config or identity lost after firmware upgrade | Unlikely with the official package (conffiles are preserved) — check third-party install methods instead | Confirm `/etc/config/tailscale` and `/etc/config/tailscaled.state` survived; if you used a third-party UPX-compressed build, check that project's own persistence notes |

---

## Sources

- Tailscale CLI reference — `tailscale up` flags
  (tailscale.com/docs/reference/tailscale-cli/up, tailscale.com/kb/1080/cli)
- Tailscale KB — Subnet routers (tailscale.com/kb/1019/subnets)
- Tailscale KB — Exit nodes quick guide (tailscale.com/kb/1408/quick-guide-exit-nodes)
- Tailscale KB — Site-to-site networking (tailscale.com/kb/1214/site-to-site)
- OpenWrt package feed (`net/tailscale`, `luci-app-tailscale`,
  `luci-app-tailscale-community`, `luci-app-tailscale-ng`)
- The `ip6tables-nft`/firewall4 gap confirmed independently on this same HX21 while
  migrating to NetBird (see `docs/vpn-netbird.md`) — same underlying platform issue
