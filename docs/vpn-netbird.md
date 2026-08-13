# NetBird on OpenWrt (Imou HX21 / MediaTek MT7981B)

A complete, tested walkthrough for installing NetBird on an OpenWrt router, using it
as a routing peer for LAN access, and optionally as an internet exit node.

Written against NetBird's official OpenWrt package feed (docs updated 2026-07-06)
and OpenWrt's `uci`/`opkg`/`apk` tooling. Verify your own OpenWrt release and NetBird
version against the table below before running anything — package managers and
NetBird versions differ across OpenWrt releases.

---

## 0. Check your OpenWrt release first

```sh
cat /etc/openwrt_release
```

This determines which package manager and which NetBird version you'll get from the
feed:

| OpenWrt release  | NetBird version (feed) | Package manager |
|-------------------|------------------------|------------------|
| 23.05              | 0.24.x                  | `opkg`            |
| 24.10              | 0.59.x                  | `opkg`            |
| 25.12 and later    | 0.66.x                  | `apk`             |

> OpenWrt switched its default package manager from `opkg` to `apk` starting with
> 24.x snapshots / stabilizing in 25.x. Use the column above to know which one your
> router has — do not assume.

### Prerequisites

- SSH access to the router (`ssh root@<router-ip>`)
- Enough free flash storage — the NetBird binary is relatively large; low-storage
  routers (small NOR flash devices) may not have room. Check with `df -h`.
- A **setup key** generated from the NetBird dashboard (Team → Setup Keys), or a
  self-hosted NetBird management server URL if you're not using NetBird Cloud.

> The official one-line install script at `pkgs.netbird.io/install.sh` **does not
> support OpenWrt**. Only the OpenWrt package feed method below works.

---

## 1. Install the NetBird package

### OpenWrt 24.10 and earlier (`opkg`)

```sh
opkg update
opkg install netbird
```

### OpenWrt 25.12 and later (`apk`)

```sh
apk update
apk add netbird
```

Both install a `procd` init script at `/etc/init.d/netbird`.

---

## 2. Enable and start the service

```sh
/etc/init.d/netbird enable
/etc/init.d/netbird start
```

At this point the daemon is running but not yet authenticated to any NetBird
network — it's waiting for a login.

---

## 3. Connect the router to your NetBird network

### NetBird Cloud, with a setup key

```sh
netbird up --setup-key <SETUP_KEY>
```

### Self-hosted NetBird

```sh
netbird up --setup-key <SETUP_KEY> --management-url https://netbird.example.com:443
```

Replace the URL with your own management server's address.

### Verify

```sh
netbird status -d
```

`-d` gives the detailed/debug view — confirm `Management: Connected` and
`Signal: Connected`, and that a peer IP has been assigned.

Once connected, the router shows up on the **Peers** page of your NetBird dashboard.
NetBird creates and fully manages its own WireGuard interface named **`wt0`**.

> **Do not** touch `wt0` through OpenWrt's LuCI WireGuard UI or manual `uci` WireGuard
> config. NetBird owns that interface and its keys entirely — manual edits will
> conflict with it and can break the connection.

---

## 4. DNS configuration (optional, only if you use NetBird DNS features)

OpenWrt's `dnsmasq` already listens on port 53, which conflicts with NetBird's own
managed DNS resolver. If you want NetBird DNS features (domain resources, peer name
resolution via MagicDNS-equivalent), you need to run NetBird's resolver on an
alternate port and forward the relevant domains to it from dnsmasq.

If you don't need domain resources / peer-name resolution, **skip this section
entirely** — plain connectivity works without it.

### 4.1 Pin NetBird's DNS resolver to an alternate port

```sh
netbird up --dns-resolver-address 127.0.0.1:5053
```

(NetBird auto-falls-back to another port if 53 is taken anyway, but pinning it keeps
the dnsmasq forwarding rule below valid and predictable.)

### 4.2 Forward NetBird's DNS domain to that port via dnsmasq

Replace `netbird.cloud` with `netbird.selfhosted` (or your custom DNS domain) if
you're self-hosting:

```sh
uci add_list dhcp.@dnsmasq[0].server='/netbird.cloud/127.0.0.1#5053'
uci commit dhcp
/etc/init.d/dnsmasq restart
```

---

## 5. Use the router as a routing peer (LAN access from other NetBird peers)

By default, the router is just a normal client peer — nothing else on your LAN can
be reached through it, and it doesn't route anyone else's traffic. To let other
NetBird peers reach devices on the router's LAN (or vice versa), configure it as a
**routing peer**.

### 5.1 Register the `wt0` interface in OpenWrt's network config

```sh
uci set network.netbird=interface
uci set network.netbird.proto='none'
uci set network.netbird.device='wt0'
uci commit network
```

### 5.2 Create a firewall zone and allow forwarding

```sh
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
> exists, don't add a duplicate — you'll hit the same
> `redefinition of symbol` nftables error you'd get from duplicating any zone name.
> Check first with:
> ```sh
> uci show firewall | grep -i netbird
> ```

### 5.3 Advertise your LAN as a network resource

In the NetBird dashboard: **Networks → Add Network Resource**, add the router's LAN
subnet (e.g. `192.168.1.0/24`), and set this router as the routing peer. Then create
an access-control policy granting the groups you want access to that resource.

---

## 6. Use the router as an internet exit node (route your traffic through the router's WAN IP)

This is the "make my router's IP browse the internet" use case — all traffic from
your other devices (phone, laptop) egresses through the router's WAN connection and
appears with the router's public IP.

Requires **NetBird v0.27.0+** (irrelevant here since the router only needs to *be*
the exit node — the version requirement is what enforces default-route + masquerade
support server-side).

### 6.1 In the NetBird dashboard

1. Go to **Peers**, find the router (it should already be listed and connected from
   step 3).
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

### 6.2 On your phone

Open the NetBird app → select the router as the exit node (or confirm it applied
automatically if Auto Apply was left on).

### 6.3 Verify

From the phone, check `https://ifconfig.me` (or similar) — it should now show the
router's WAN IP.

### 6.4 Known caveat — LAN exposure through exit nodes

There's an open upstream issue (netbirdio/netbird #5797) where devices using a
router as an exit node can reach the **entire LAN subnet** of that router — not just
the internet — even with a custom ACL policy and no explicit LAN route advertised.
This happens because the exit node's masquerade rule forwards everything arriving on
`wt0` to the WAN interface *and* the LAN forwarding path is not restricted at the
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

## 7. Persistence across reboots and firmware upgrades

The OpenWrt package marks NetBird's config as a conffile, so it survives
`sysupgrade` without extra steps. Storage location depends on your package version:

- OpenWrt 25.12+: `/root/.config/netbird`
- OpenWrt 24.10 and earlier: `/etc/netbird/config.json`

Both paths are included automatically in the `sysupgrade` config backup.

---

## 8. Quick troubleshooting reference

| Symptom | Likely cause | Check |
|---|---|---|
| `netbird up` hangs or fails to authenticate | Setup key expired/already used, or wrong `--management-url` | Generate a fresh one-time or reusable key in the dashboard; confirm URL if self-hosted |
| Peer shows in dashboard but no traffic flows | Firewall zone/forwarding not applied, or `wt0` not in a zone at all | `uci show firewall \| grep -i netbird`, `ip a show wt0` |
| Duplicate zone / `redefinition of symbol` on firewall restart | Package auto-created a `netbird` zone and you added a second one manually | `uci show firewall`, delete the duplicate `cfgXXXXXX` entry, not the package-managed one |
| DNS-based domain resources not resolving | Port 53 conflict with dnsmasq never resolved | Complete section 4, confirm with `netbird status -d` that a DNS resolver line is listed |
| Exit node traffic reaches router's LAN devices | Known upstream issue #5797 | Apply the access-control-group mitigation in section 6.4 and test manually |
| Config lost after firmware upgrade | Wrong path assumption for your OpenWrt version | Confirm which path in section 7 matches your release before upgrading |

---

## Notes on why this differs from Tailscale on the same router

- NetBird creates `wt0`; Tailscale creates `tailscale0` — don't mix firewall rules
  between the two if you're running both for comparison; they need separate zones.
- NetBird's exit-node ACL model is enforced primarily through **routes + access
  control groups** in the dashboard, not through the OpenWrt firewall itself — the
  local `uci` firewall rules here exist only to stop OpenWrt from blocking the
  tunnel interface, not to enforce NetBird-level policy.
- Unlike Tailscale's OpenWrt package (which is community/upstream OpenWrt-maintained
  and has had longer field exposure on embedded targets), NetBird's own docs note
  x86_64 as the most tested platform, with arm64/other architectures "supported but
  less tested." The Imou HX21 (MT7981B, arm64) falls in that less-tested bucket —
  keep an eye on `dmesg` and `logread` after installation for anything unusual.

---

## Sources

- NetBird OpenWrt Installation docs (docs.netbird.io/get-started/install/openwrt)
- NetBird Exit Node configuration docs (docs.netbird.io/manage/network-routes/use-cases/exit-nodes)
- NetBird "How Routing Peers Work" docs (docs.netbird.io/manage/networks/how-routing-peers-work)
- netbirdio/netbird GitHub issue #5797 (LAN subnet leak through exit nodes)