# Multi-Tier Virtual Homelab

A segmented network lab built on Debian using Incus containers and an Alpine Linux router. The lab is designed as a **three-legged firewall** with four internal subnets, a single enforcement point, and WireGuard VPN as the only inbound path.

Built and maintained as a personal learning environment for networking, firewall design, and service deployment fundamentals.

## Topology

![Network topology](topology.png)

```
Internet
   |
Debian Host (10.10.10.1) ── the lab's upstream
   |
Alpine Router Container (10.10.10.62) ── lab edge
   |
   ├── eth1 ── Testbed A     (10.0.1.0/24)
   ├── eth2 ── Testbed B     (10.0.2.0/24)
   ├── eth3 ── App tier      (10.0.20.0/24) ── nextcloud (10.0.20.17)
   └── eth4 ── DB tier       (10.0.30.0/24) ── postgres  (10.0.30.13)
```

From the lab's perspective, the Debian host *is* the internet. The router container's `eth0` faces Debian and MASQUERADEs all outbound traffic through it. Whatever sits upstream of Debian (home router, ISP) is abstracted away.

## Host and Runtime

| | |
|---|---|
| Host | Dell Latitude, i5-8365U, 32 GB RAM, 256 GB SSD |
| OS | Debian (bare metal) |
| Container runtime | Incus |
| Containers | 5 (router, nextcloud, postgres, server, client) |
| Internal networks | 4 |

The lab runs entirely in Incus containers. No VMs, no Docker.

## Architecture Note

This is structurally DMZ-shaped but enforced as a **three-legged firewall**: one router holds all inter-zone rules. A true DMZ requires two enforcement points so that compromise of one layer still leaves another. Adding a second enforcement boundary is on the roadmap.

## Networking

**Routing.** Static only. No dynamic routing protocols. Single router, so OSPF or BGP would add no value here.

**DHCP and DNS.** Both handled by `dnsmasq` on the router.

- DHCP serves eth1 through eth4. Uplink (eth0) is excluded with `no-dhcp-interface=eth0`.
- Pool: `.10` to `.20` of each `/24`. 24-hour leases.
- Gateway (option 3) and DNS (option 6) pushed as the router's IP on each segment.
- DNS is authoritative for `.local`:
  - `nextcloud.local` → `10.0.20.17`
  - `postgres.local` → `10.0.30.13`
- Upstream DNS chain: container → router dnsmasq → Debian host → ISP DNS.

**NAT.** Outbound `MASQUERADE` on `eth0` only. No inbound port forwards from the home LAN. The only inbound path into the lab is WireGuard.

**IPv6.** Incus assigns ULA addresses (`fd42::/64`) to every container, but no `ip6tables` rules are configured. The v6 plane is currently unenforced — flagged as a gap.

## Firewall

Default policies:

- `INPUT`: ACCEPT
- `OUTPUT`: ACCEPT
- `FORWARD`: **DROP** — this is the segmentation

FORWARD ACCEPT rules:

```
Chain FORWARD (policy DROP)
target  prot  in    out   source           destination  state
ACCEPT  all   wg0   eth3  10.200.0.0/24    0.0.0.0/0
ACCEPT  all   *     *     10.0.0.0/16      0.0.0.0/0
ACCEPT  all   eth0  *     0.0.0.0/0        0.0.0.0/0    RELATED,ESTABLISHED
```

**What this gives:** nothing upstream of the router (including the Debian host itself) can initiate inbound connections to any internal segment. The only inbound path is the WireGuard tunnel.

**What this doesn't give:** internal-to-internal traffic is trusted via the blanket `10.0.0.0/16 → anywhere` rule. Nextcloud reaches Postgres on TCP/5432 through this rule. Per-port tightening between zones is the next obvious step.

## Services

**Nextcloud** (app tier, 10.0.20.17)

- Apache 2.4 on Debian
- DocumentRoot: `/var/www/html/nextcloud`
- Listens on port 443 only — port 80 is closed (see debugging story 1)
- Self-signed cert at `/etc/ssl/certs/nextcloud.crt`
- `overwriteprotocol=https`, `trusted_domains` set

**PostgreSQL** (DB tier, 10.0.30.13)

- Dedicated container on a separate segment from the app
- Used only by Nextcloud
- Cross-segment query path: `eth3 → router → eth4`

**WireGuard** (router)

- UDP/51820
- Tunnel network: `10.200.0.0/24`, router at `10.200.0.1`
- One peer (laptop) at `10.200.0.2`
- Explicit per-interface ACCEPT rules grant the WG client reach into all four internal segments

## Debugging Stories

The most valuable part of running a homelab is what breaks. Three real ones:

### 1. WireGuard "encryption" was not protecting credentials

**Symptom:** A Wireshark capture on the router's `wg0` interface during a Nextcloud login showed the password in cleartext.

**Root cause:** WireGuard encrypts the tunnel hop only (laptop ↔ router). Once packets are decrypted at `wg0`, they continue as normal IP. Nextcloud was being accessed over HTTP, so the application layer was never encrypted. I had assumed VPN = end-to-end protection. It is not.

**Fix:** Removed `Listen 80` from Apache's `ports.conf`, deleted the `*:80` vhost from `nextcloud.conf`, verified only 443 listening via `ss -tlnp`. Re-captured. Payload was TLS, password no longer visible.

**Lesson:** Network-layer encryption is not end-to-end. The wire is not the trust boundary — the application is. Always run TLS at the app layer regardless of what's happening below it.

### 2. WireGuard peer silently dropped from runtime config

**Symptom:** `wg show` reported the interface up and listening on UDP/51820 but no peer registered, despite a `[Peer]` block in the config.

**Root cause:** Two bugs at once.

1. The client public key in the config was identical to the server's own public key — a key-generation copy-paste error. WireGuard silently refuses to register a peer whose pubkey matches the interface.
2. The directive was written as `AllowedIps` instead of `AllowedIPs`. `wg-quick` is case-sensitive and silently ignores unknown keys. A peer with no `AllowedIPs` is invalid and gets dropped.

**Fix:** Regenerated all four keys using separate `wg genkey > file` and `wg pubkey < file > pubfile` calls. Verified the four key strings were distinct. Corrected the casing.

**Lesson:** When `wg show` is silent about a peer that should exist, the failure is parsing, not networking. First check: does the interface pubkey shown by `wg show` differ from the configured peer pubkey? Second check: case-sensitive directive names.

### 3. Locked myself out by closing port 80

**Symptom:** Typing `10.0.20.17` into a browser failed with "site can't be reached", right after I'd successfully tightened the Apache config.

**Root cause:** Browsers default to `http://` for bare-IP URLs. Apache was not broken — it was correctly refusing connections on a port it no longer listened on.

**Fix:** Enabled HTTPS-First mode in the browser. Bookmarked `https://10.0.20.17` explicitly.

**Lesson:** Removing services has UX consequences. When tightening security, the listening-port surface is not the whole picture — the user's first interaction with the new state (bookmarks, browser autocomplete, default URL schemes) matters too.

## Roadmap

Committed:

- **Monitoring** — Prometheus and Grafana, probably on one of the unused testbed containers

Candidate list (not committed):

- Internal CA via `step-ca` to replace self-signed certs
- IPv6 firewall rules
- Per-port internal segmentation (tighten the blanket `10.0.0.0/16` rule)
- Suricata IDS on the router
- Hostname migration from `.local` to `.lan`
- Automated backups for Nextcloud and Postgres
- Second WireGuard client for mobile
- Second enforcement point to convert the three-legged firewall into a true DMZ

## Tech Stack

`Debian` · `Incus` · `Alpine Linux` · `iptables` · `dnsmasq` · `WireGuard` · `Apache` · `Nextcloud` · `PostgreSQL` · `OpenSSL`

---

Maintained by Rudra Patel. Built as a learning environment, documented honestly.