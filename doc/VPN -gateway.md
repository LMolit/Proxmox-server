# VPN Gateway (CT 106)

## Overview

Dedicated WireGuard routing gateway. Provides a network-level kill switch
for qBittorrent (CT 107): qBittorrent has no default route to the real
internet at all, and instead routes exclusively through this container's
`wg0` tunnel. If the tunnel drops, qBittorrent loses all internet access —
there is no fallback path, by network design rather than firewall policy
alone. Built manually (not via helper script) due to the non-standard
dual-interface routing requirement.

VPN provider: Proton VPN (paid). NAT-PMP/port forwarding deliberately
**not** enabled (known to be unreliable over WireGuard) — revisit later
if desired.

## Specs

| Field | Value |
|---|---|
| CT ID | 106 |
| Hostname | vpn-gateway |
| Template | Debian 12 (standard) |
| Unprivileged | Yes |
| Cores | 1 |
| RAM | 512MB |
| Disk | 4GB |
| onboot | 1 |

## Network

| Interface | Bridge | IP | Gateway | Purpose |
|---|---|---|---|---|
| eth0 | vmbr0 (LAN) | 10.0.0.15/24 | 10.0.0.1 | Real internet access — needed for the WireGuard handshake to Proton's servers |
| eth1 | vmbr1 (isolated, no physical NIC) | 10.10.10.1/24 | *(none)* | Faces qBittorrent only; this side deliberately has no default route |
| wg0 | — | 10.2.0.2/32 (Proton-assigned) | — | The actual VPN tunnel interface |

**vmbr1** has `bridge-ports none` — it is not attached to any physical NIC
and has no IP on the Proxmox host itself. It exists purely as an isolated
virtual switch connecting this container and qBittorrent.

## Key files & paths

| Path | Purpose |
|---|---|
| `/etc/wireguard/wg0.conf` | WireGuard config (Proton-generated + custom PostUp/PostDown rules). **Contains a private key — never commit to git.** Permissions locked to `600`. |
| `/etc/pve/lxc/106.conf` (host-side) | Container config; includes the TUN device cgroup allow rule and mount entry |
| `/etc/sysctl.conf` | Contains `net.ipv4.ip_forward=1` (persisted) |

## Ports

| Port | Protocol | Purpose |
|---|---|---|
| 51820 (outbound only) | UDP | WireGuard handshake to Proton's endpoint (`138.199.50.149` for the currently-configured peer — will differ if the config is regenerated against a different server) |

No inbound ports are exposed by this container — it has no services
listening for LAN connections; it's purely a router.

## Firewall / routing rules (in `wg0.conf`, applied via PostUp/PostDown)

```
iptables -t nat -A POSTROUTING -o wg0 -j MASQUERADE
iptables -A FORWARD -i eth1 -o wg0 -j ACCEPT
iptables -A FORWARD -i wg0 -o eth1 -m state --state RELATED,ESTABLISHED -j ACCEPT
iptables -A FORWARD -i eth1 -o eth0 -j DROP
```

The last rule is a deliberate belt-and-suspenders kill-switch backstop —
`eth1` already has no default route, so this traffic is structurally
impossible anyway, but the explicit `DROP` protects against future
misconfiguration (e.g. someone adding a gateway to `eth1` later without
realizing what this container is for).

## Quick tests

```bash
# Is the tunnel up and recently handshaking?
pct exec 106 -- wg show

# Does this container's own traffic actually route through Proton?
pct exec 106 -- curl https://am.i.mullvad.net/ip
# Compare against the real host IP (run from Proxmox host directly, not the container):
curl https://am.i.mullvad.net/ip

# Confirm the IP above belongs to Proton, not a residential ISP:
pct exec 106 -- curl https://ipinfo.io/<ip-from-above>/org

# Is IP forwarding actually enabled?
pct exec 106 -- cat /proc/sys/net/ipv4/ip_forward   # should print 1

# Is the tunnel set to persist across reboots?
pct exec 106 -- systemctl is-enabled wg-quick@wg0   # should print "enabled"
```

## Kill-switch verification (run periodically, especially after any config change)

```bash
# Simulate the tunnel dropping:
pct exec 106 -- wg-quick down wg0

# From qBittorrent's container (once CT 107 exists), confirm total loss
# of internet access — not a fallback to a direct connection:
pct exec 107 -- curl --max-time 5 https://am.i.mullvad.net/ip
# Expect this to time out / fail entirely.

# Bring the tunnel back up:
pct exec 106 -- wg-quick up wg0
```

## Troubleshooting

- **`wg-quick up wg0` fails with `resolvconf: command not found`:**
  `apt install -y resolvconf` inside the container.
- **No handshake / `wg show` shows no recent `latest handshake`:** check
  `eth0`'s route to the internet (`pct exec 106 -- ping -c3 1.1.1.1`);
  confirm Proton's endpoint IP hasn't changed (regenerate config if the
  server was decommissioned).
- **qBittorrent has no internet even with tunnel up:** verify the
  FORWARD/MASQUERADE rules are actually present (`pct exec 106 --
  iptables -L -v -n` and `iptables -t nat -L -v -n`) — a `wg-quick down`
  followed by `up` should reapply them from `wg0.conf`, but a manual
  `iptables -F` elsewhere would wipe them silently.

## Future improvements

- Enable NAT-PMP/port forwarding once the core stack is stable, for
  improved torrent swarm connectivity (inbound peer connections).
- Consider a monitoring/alerting check (e.g. a periodic cron job) that
  verifies `wg show` shows a recent handshake and alerts if the tunnel
  has silently dropped without qBittorrent's kill switch being the cause.
- Store the Proton config's expiration/rotation reminder somewhere
  (Proton WireGuard configs can expire and need regeneration).

## Changelog

- *(date)* — Container built manually, tunnel established and verified
  against Proton AG (AS208172), kill-switch rules applied, persistence
  enabled via `systemctl enable wg-quick@wg0`.
