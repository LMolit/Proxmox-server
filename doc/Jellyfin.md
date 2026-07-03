# Jellyfin (CT 101)

## Overview

Media streaming server. Serves Movies, TV, and (planned) Audiobooks/Books to
LAN clients. Installed via the Proxmox Community Helper Script, running as an
unprivileged LXC. Hardware-accelerated transcoding via Intel Quick Sync
(iGPU passthrough), verified working with `vainfo` prior to this
documentation being written.

## Specs

| Field | Value |
|---|---|
| CT ID | 101 |
| Hostname | jellyfin |
| Template | *(TBD — fill in from `pct config 101`)* |
| Unprivileged | Yes |
| Cores | *(TBD)* |
| RAM | *(TBD)* |
| Disk | *(TBD)* |

## Network

| Interface | Bridge | IP | Gateway |
|---|---|---|---|
| eth0 | vmbr0 (LAN) | 10.0.0.10/24 | 10.0.0.1 |

## Key files & paths

| Path (in container) | Purpose |
|---|---|
| `/media` | Bind mount → host `/mnt/media` (mergerfs pool) |
| `/media/Movies` | Movie library |
| `/media/TV` | TV library |
| `/media/Audiobooks` | Audiobook library (planned — not yet added to Jellyfin libraries) |
| *(TBD)* | Jellyfin config/metadata directory — confirm helper script's default location |

**Host-side ownership note:** everything under `/mnt/media` is owned by
`mediauser` (UID/GID 1000:1000) on the Proxmox host. Jellyfin's internal
user/UID mapping should be verified against this — unconfirmed as of this
doc. If Jellyfin can read but not write (e.g. for metadata `.nfo`
generation or thumbnail extraction inside library folders), this mapping
is the first thing to check.

## Ports

| Port | Protocol | Purpose |
|---|---|---|
| 8096 | TCP | Web UI / API (HTTP) |
| 8920 | TCP | Web UI / API (HTTPS, if enabled) |
| 1900 | UDP | DLNA discovery (if enabled) |
| 7359 | UDP | Jellyfin client auto-discovery |

*(Confirm which of these are actually enabled/needed for your setup —
DLNA/auto-discovery may be unused if you're not running local network
client auto-detection.)*

## Quick tests

```bash
# Is the service running?
pct exec 101 -- systemctl status jellyfin

# Is hardware transcoding available inside the container?
pct exec 101 -- vainfo

# Is the media mount visible and populated?
pct exec 101 -- ls -la /media

# Can Jellyfin's process actually write inside the media mount?
# (run as whatever UID Jellyfin runs as inside the container)
pct exec 101 -- touch /media/Movies/.write_test && pct exec 101 -- rm /media/Movies/.write_test
```

## Troubleshooting

- **Library scan not picking up new files:** confirm `/media` bind mount
  is live (`pct exec 101 -- mount | grep media`), and check file
  permissions on the host under `/mnt/media`.
- **Transcoding falls back to software/CPU:** re-run `vainfo` inside the
  container; if it errors, check that `/dev/dri` is still passed through
  correctly in `pct config 101` (helper script should have handled this,
  but worth re-verifying after any Proxmox kernel update).
- **Container won't start after host reboot:** check `pct config 101` for
  `onboot: 1`; check `journalctl -xe` on the host around the container's
  start time.

## Future improvements

- Add Audiobooks and Books libraries once Readarr/Audiobookshelf are live.
- Confirm and document the exact UID Jellyfin runs as internally, and
  align it explicitly with `mediauser` (1000:1000) rather than assuming
  the helper script already did this correctly.
- Consider moving to a longer library-refresh interval or webhook-based
  refresh once Sonarr/Radarr are live, rather than polling.
- Document backup approach for Jellyfin's config/metadata directory
  (separate from media itself, per the stack-wide backup strategy).

## Changelog

- *(date)* — Initial documentation written. Container pre-existed this
  documentation effort; some fields marked TBD pending verification.
