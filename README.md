# Multi-VLAN OPNsense + Dual Pi-hole DNS Setup (with Stable IPv6)

This guide walks through setting up network-wide, force-enforced Pi-hole DNS filtering across multiple VLANs on OPNsense, including full IPv6 support that won't break every time your ISP reshuffles your delegated prefix.

## What you end up with

- Every VLAN's DNS traffic transparently forced through Pi-hole (devices can't bypass it by setting their own DNS server)
- Two Pi-hole instances for redundancy
- Full IPv4 + IPv6 support
- Pi-hole addressed on a permanent internal IPv6 range (ULA) so nothing breaks when your ISP's delegated prefix changes

## Prerequisites

- OPNsense already installed, with your VLANs created as separate interfaces (this guide assumes 5: LAN + 4 others, e.g. IoT, Guest, Trusted, Cameras — adjust to your own setup)
- Two Pi-hole instances running (VMs, containers, or physical) reachable on your network
- Your ISP supports IPv6 with DHCPv6-PD (most cable/fiber ISPs do — ask support or check their subscriber FAQ for "prefix delegation size" if unsure)

---

## Part 1 — IPv6 Prefix Delegation on WAN

This gives your router (and each VLAN) a real, routable IPv6 subnet from your ISP.

1. Interfaces → WAN.
2. IPv6 Configuration Type: DHCPv6.
3. Under DHCPv6 Client Configuration, set Prefix Delegation size to whatever your ISP delegates (commonly `/60`, sometimes `/56` — check with your provider if unsure).
4. Check "Send IPv6 prefix hint." Some ISPs silently ignore your requested size without this.
5. Save & Apply.
6. Check Interfaces → Overview — WAN should now show a delegated block (e.g. `/60`), not just a single address.

## Part 2 — Give Each VLAN Its Own /64

For every interface (LAN + each VLAN):

1. Interfaces → `<interface>`.
2. IPv6 Configuration Type: Track Interface (may show as "Track Interface (legacy)" depending on version).
3. Track IPv6 Interface: WAN.
4. IPv6 Prefix ID: a unique number per interface (e.g. LAN=0, VLAN1=1, VLAN2=2, etc. — any numbering works as long as each is unique; matching them to your subnet numbering is just for readability).
5. Save. Repeat for each interface, then Apply.

## Part 3 — Give Pi-hole a Permanent Address (ULA)

Don't point anything at your Pi-holes' ISP-assigned IPv6 addresses — those change whenever your delegated prefix does, silently breaking every rule that references them.

Instead, give Pi-hole a Unique Local Address (ULA) — a private IPv6 range you control forever, independent of your ISP.

1. Generate a random `/48` ULA prefix (RFC 4193 compliant — many online generators exist, or use any tool that produces an `fd00::/8` block with proper random entropy). You'll get something like `fdXX:XXXX:XXXX::/48`.
2. OPNsense: Interfaces → Virtual IPs → Add → Mode: IP Alias, Interface: LAN, Network/Address: `<your-ula-prefix>::1/64`. Save & apply. This makes the `/64` routable on your LAN.
3. On each Pi-hole, add a static IPv6 address in that range alongside its existing config. On Debian/Ubuntu with ifupdown (`/etc/network/interfaces`):

   ```
   iface eth0 inet static
       address 10.0.0.2/24          # your existing IPv4 line
       address <your-ula-prefix>::2/64
       gateway 10.0.0.1
   ```

   Apply with `sudo ifdown eth0 && sudo ifup eth0`. Use `::3` for Pi-hole 2.

4. Verify: `ip -6 addr show eth0` — you should see the ULA address with `valid_lft forever`.
5. Use these two ULA addresses in the `PiHole_DNS_IPv6` alias from Part 4.

> **Proxmox note:** if running on Proxmox, the local IPv6 address can be overwritten when the LXC reboots. To stop this, navigate to Proxmox Node → PiHole LXC → Network tab → select the Network Interface → Edit → under the IPv6/CIDR section → add your Local IPv6 address → Save/Apply.

## Part 4 — Firewall Aliases

Firewall → Aliases → Add these:

| Name | Type | Content |
|---|---|---|
| `PiHole_DNS` | Host(s) | Your two Pi-hole IPv4 addresses |
| `PiHole_DNS_IPv6` | Host(s) | Your two Pi-hole IPv6 addresses (see Part 3 — use the permanent ULA ones, not ISP-derived) |
| `PrivateNetworks` | Network(s) | `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16` |
| `PrivateNetworks_IPv6` | Network(s) | `fc00::/7`, `fe80::/10` |
| `LAN_Restricted_v4` | Network(s) | `10.0.10.0/24`, `10.0.40.0/24` (Trusted & Cameras subnets) |
| `LAN_Restricted_v6` | Network(s) | `_opt5_network`, `opt6_network` (Trusted & Cameras VLAN interfaces) |

Add a `_Restricted` alias per VLAN if you want to block specific VLANs from reaching each other (e.g. general LAN devices shouldn't reach security cameras).

## Part 5 — Router Advertisements (per VLAN)

For each VLAN you want handing out DNS over IPv6 (Services → Router Advertisements):

1. Select the interface.
2. Mode: **Assisted** — lets clients self-address via SLAAC and read DNS straight from the advertisement.
3. DNS Servers (RDNSS field): your two Pi-hole Local IPv6 ULA addresses from Part 3.
4. Save & Apply.

## Part 6 — Force DNS via NAT Redirect (per VLAN)

This transparently redirects any DNS query not already going to Pi-hole, so devices can't bypass it by hardcoding a different DNS server.

Firewall → NAT → Port Forward, for each VLAN, create both an IPv4 and IPv6 version:

| Field | Value |
|---|---|
| Interface | the VLAN |
| Protocol | TCP/UDP |
| Source | VLAN network |
| Destination | Invert: enabled, Destination: the `PiHole_DNS` / `PiHole_DNS_IPv6` alias |
| Destination port | DNS (53) |
| Redirect target IP | One of your Pi-hole addresses (v4: an actual IP; v6: the Local ULA address) |
| Redirect target port | 53 |
| Filter rule association | Pass (auto-creates the matching filter rule) |

**Critical step — don't skip this:** Firewall → Settings → Advanced → under Network Address Translation, check "Reflection for destination NAT." Without this, devices on the same subnet as Pi-hole will get DNS replies that appear to come from an unexpected address and get silently dropped by the client OS — this presents as random, hard-to-diagnose "no internet" on some devices and not others.

### Exclude Pi-hole's own queries on whichever VLAN hosts it

If one of your Pi-holes lives on the same VLAN as one of these redirect rules (commonly LAN), that rule's "Source: VLAN network" also matches the Pi-hole host itself. That means Pi-hole's own outbound queries to its upstream resolver get caught by its own rule and redirected right back to itself — a self-redirect loop. Pi-hole can never reach upstream, so it can only answer from cache, and every VLAN that depends on it for DNS starts failing intermittently (not just the VLAN hosting Pi-hole).

**Fix:** on the redirect rule for the VLAN where Pi-hole physically resides, set Source to the same `PiHole_DNS` / `PiHole_DNS_IPv6` alias used in the Destination field, and enable Invert Source too. This reads as "traffic from this VLAN, except from Pi-hole itself" — so Pi-hole's own upstream queries pass straight through instead of looping. VLANs that don't host a Pi-hole don't need this change, since their redirect rule's source never overlaps with a Pi-hole's address.

| Field | Value |
|---|---|
| Interface | LAN |
| Protocol | TCP/UDP |
| Invert Source | Enabled |
| Source | `PiHole_DNS` / `PiHole_DNS_IPv6` alias |
| Source Port | Any |
| Invert Destination | Enabled |
| Destination | `PiHole_DNS` / `PiHole_DNS_IPv6` alias |
| Destination port | DNS (53) |
| Redirect target IP | One of your Pi-hole addresses (v4: an actual IP; v6: the Local ULA address) |
| Redirect target port | 53 |
| Filter rule association | Pass (auto-creates the matching filter rule) |

## Part 7 — Test

1. Enable NAT rules only first (leave the block-DNS rules off). Test each VLAN's devices.
2. If that's stable, enable the block-DNS rules and test again.
3. Testing both changes at once makes it hard to tell which change broke what if something does — change one thing at a time.

## Troubleshooting

**A device shows "No Internet" on Wi-Fi despite actually having a working connection (common on iPhones/Apple TVs):**
This is usually Apple's own captive-network check failing, not a real outage — it happens when Pi-hole's blocklists catch one of Apple's connectivity-check domains (`captive.apple.com` and similar). Check Pi-hole's Query Log for blocked queries from the affected device and whitelist anything Apple-related you find blocked.

**A VLAN's devices lose internet specifically when the NAT redirect is enabled:**
Almost always the NAT reflection setting from Part 6 — go check it's enabled.

**DNS breaks network-wide (every VLAN, not just one) after enabling the force-DNS NAT redirect:**
Check whether one of your redirect rules shares a VLAN with a Pi-hole itself — see the update in Part 6. An un-excluded source on that rule loops Pi-hole's own upstream queries back to itself, which knocks out DNS for every VLAN that relies on it, not just the one hosting it.

**IPv6 DNS randomly stops working after a while:**
Check whether you're still referencing an ISP-derived IPv6 address anywhere (aliases, NAT targets, RA lists) instead of the permanent ULA ones from Part 3 — ISP-delegated addresses can and do change.

**A VLAN's Router Advertisement mode is "Managed" instead of "Assisted":**
Only use Managed if you're actually running a DHCPv6 server on that interface. If you're not, clients will wait indefinitely for a config that never arrives — switch to Assisted.

## Part 8 — Firewall Rules Reference

The full rule set below reflects a working 5-VLAN layout (LAN + IoT + Guest + Trusted + Cameras), listed in evaluation order per interface (rules are evaluated top to bottom, first match wins). Use it as a reference for the intended traffic policy:

- LAN can reach IoT and Guest (outbound-initiated only — IoT/Guest can't initiate back), and the internet. Blocked from Trusted and Cameras.
- IoT/Guest can reach Pi-hole and the internet only — no access to any internal subnet, including LAN.
- Trusted can reach every VLAN (LAN, IoT, Guest, Cameras) and the internet — nothing can initiate back to it.
- Cameras can reach Pi-hole only — no internet, no other internal subnet. Fully isolated except for what Trusted initiates to it.

### LAN interface

| Seq | Action | IP | Source | Destination | Description |
|---|---|---|---|---|---|
| 900 | Pass | v4 | LAN | Pihole_DNS | Allow Access to PiHole DNS |
| 950 | Pass | v6 | LAN | PiHole_DNS_IPv6 | Allows Access to PiHole IPv6 DNS |
| 1200 | Pass | v4 | LAN | IoT, Guest | Allows Lan Net to Connect to IOT, Guest Devices |
| 2050 | Block | v4 | LAN | LAN_Restricted_v4 (Trusted/Cameras) | Blocks Lan to Trusted & Cameras Devices |
| 2060 | Block | v6 | LAN | LAN_Restricted_v6 (Trusted/Cameras) | Blocks Lan to Trusted & Cameras Devices IPv6 — see Part 9 |
| 2100 | Pass | v4 | LAN | any | Default Allow Lan to any Rule |
| 2200 | Pass | v6 | LAN | any | Default Allow LAN IPv6 to any Rule |

### IoT (IOT_vLan) interface — VLAN tag: 20

| Seq | Action | IP | Source | Invert Destination | Destination | Description |
|---|---|---|---|---|---|---|
| 100 | Pass | v4 | IoT | Disabled | Pihole_DNS | Allows IOT_vLan to Pi-Hole |
| 200 | Pass | v6 | IoT | Disabled | PiHole_DNS_IPv6 | Allow IOT_vLan IPv6 DNS to Pi-Hole |
| 250 | Pass | v4 | IoT | Enabled | PrivateNetworks | Allow Access to Internet & Blocks Access to Private Networks |
| 275 | Pass | v6 | IoT | Enabled | PrivateNetworks_IPv6 | Allow Access to Internet & Blocks Access to Private Networks IPv6 |

### Guest (Guest_vLan) interface — VLAN tag: 30

| Seq | Action | IP | Source | Invert Destination | Destination | Description |
|---|---|---|---|---|---|---|
| 500 | Pass | v4 | Guest | Disabled | Pihole_DNS | Allow Guest_vLan to Pi-Hole DNS |
| 600 | Pass | v6 | Guest | Disabled | PiHole_DNS_IPv6 | Allow Guest_vLan to PiHole IPv6 DNS |
| 650 | Pass | v4 | Guest | Enabled | PrivateNetworks | Allow Access to Internet & Block Access to Private Networks |
| 675 | Pass | v6 | Guest | Enabled | PrivateNetworks_IPv6 | Allow Access to Internet & Blocks Access to Private Networks IPv6 |

### Trusted (Trusted_vLan) interface — VLAN tag: 10

| Seq | Action | IP | Source | Destination | Description |
|---|---|---|---|---|---|
| 1500 | Pass | v4 | Trusted | Pihole_DNS | Allow Trusted_vLan to PiHole |
| 1600 | Pass | v6 | Trusted | PiHole_DNS_IPv6 | Allows Trusted_vLan to PiHole IPv6 DNS |
| 1850 | Pass | v4 | Trusted | LAN Network | Allows Trusted Net to Connect to Lan Devices |
| 2000 | Pass | v4 | Trusted | Cameras, Guest, IoT | Allows Trusted Net to Connect to IOT, Guest, & Cameras Devices |
| 2300 | Pass | v4 | Trusted | any | Allow Trusted to any IPv4 Rule |
| 2400 | Pass | v6 | Trusted | any | Allow Trusted to any IPv6 Rule |

### Cameras (Cameras_vLan) interface — VLAN tag: 40

| Seq | Action | IP | Source | Destination | Description |
|---|---|---|---|---|---|
| 1300 | Pass | v4 | Cameras | Pihole_DNS | Allows Cameras_vLan to Pi-Hole |
| 1350 | Pass | v6 | Cameras | PiHole_DNS_IPv6 | Allow Cameras_vLan IPv6 DNS to Pi-Hole |
| 1362 | Block | v4 | Cameras | PrivateNetworks | Blocks Access to Private Networks |
| 1368 | Block | v6 | Cameras | PrivateNetworks_IPv6 | Blocks Access to Private Networks IPv6 |
| 2500 | Block | v4 | Cameras | any | Blocks Access to Internet |
| 2600 | Block | v6 | Cameras | any | Blocks Access to Internet IPv6 |

## Part 9 — Block LAN IPv6 to Trusted/Cameras (Prefix-Stable)

The IPv4 block (LAN → LAN_Restricted_v4, sequence 2050) only covers IPv4. LAN's IPv6 catch-all (sequence 2200, "Default Allow LAN IPv6 to any") has nothing blocking Trusted or Cameras ahead of it, so an IPv6-capable device on LAN can currently reach both over v6 even though v4 access is blocked.

Don't build this as a static alias of typed-in IPv6 prefixes. Trusted and Cameras get their IPv6 `/64`s via tracked IPv6 off the WAN's DHCPv6-PD delegation (Part 2) — if the ISP reshuffles the delegated prefix, a hand-typed alias goes stale silently: the rule stops matching the real subnet and quietly stops blocking anything, with no error to tip you off.

Instead, reference the interfaces directly — OPNsense resolves an interface-based source/destination to whatever address that interface currently holds, every time the ruleset reloads, so it survives a prefix reshuffle with zero maintenance. This is the same pattern already used successfully on the Trusted interface's own rules (1900, 2000), which reference "lan" and "opt6, opt4, & opt3" directly rather than static aliases.

You don't have to pick interfaces directly inside the rule itself, though — you can wrap them in an alias instead, which is cleaner if you want a single reusable, named destination. Firewall → Aliases lists an "Internal (automatic)" alias for every interface already — `__lan_network`, `__opt3_network`, `__opt4_network`, `__opt5_network`, `__opt6_network`, etc. — and those track the interface's live address exactly the same way a direct interface selection in a rule does. Build a Network(s)-type alias named `LAN_Restricted_v6`, and set its Content to `__opt5_network` (Trusted interface) and `__opt6_network` (Cameras interface). Same prefix-stable behavior, reusable by name.

To add it:

1. Firewall → Aliases → Add.
2. Name: `LAN_Restricted_v6`.
3. Type: Network(s).
4. Content: `__opt5_network`, `__opt6_network` (the automatic interface-tracking aliases for Trusted and Cameras).
5. Description: "Trusted + Cameras IPv6 Subnets - Lan Should not reach." Save.
6. Firewall → Rules → LAN → Add.
7. Action: Block.
8. IP Version: IPv6.
9. Source: LAN net.
10. Destination: `LAN_Restricted_v6`.
11. Description: "Blocks Lan to Trusted, Cameras Devices IPv6."
12. Save, then drag it into position between the existing 2050 (LAN_Restricted_v4 block) rule and 2200 ("Default Allow LAN IPv6 to any") — it must evaluate before the v6 catch-all or the catch-all will pass the traffic first.
13. Apply.

Once in place, the LAN row in the Part 8 reference table above is complete for both protocols, and the same LAN → IoT/Guest-yes, LAN → Trusted/Cameras-no, Trusted → everything-yes policy holds identically whether the ISP's delegated prefix changes or not.
