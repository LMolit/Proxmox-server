# Jellyfin + Arr stack setup

**Containers**
| Service | CT ID | IP |
|---|---|---|
| Jellyfin | 101 | 10.0.0.10 |
| Seer | 102 | 10.0.0.14 |
| Prowlarr | 103 | 10.0.0.19 |
| Sonarr | 104 | 10.0.0.18 |
| Radarr | 105 | 10.0.0.17 |
| VPN Gateway | 106 | 10.0.0.15 |
| qBittorrent | 107 | 10.0.0.16 |

I used Proxmox VE helper scripts to set each service up in an unprivileged LXC. they each have an assigned static ip address. I have edited my router's DHCP to start at 10.0.0.100 so everything between 10.0.0.2-99 is available for container static ips. ‘’

## Storage

Starting this project, I only had a 1TB HDD, and knowing I would soon be adding more, just not sure how many, I decided to create a mergerfs pool. That way my first hard drive could just be the first leaf, and I could add others later without restructuring anything. 

I then bind mounted this pool into each LXC that needed access to it, Sonarr, Radarr, Jellyfin, and qBittorrent but because I also needed these containers to write to it, and they were all unprivileged with the pool living on the Proxmox host, I ran into a UID mismatch problem. Unprivileged containers shift their internal UIDs to a high range on the host, so a container process wouldn't naturally own anything on the host-side pool. 

I solved this by creating one user, mediauser, that owns everything under /mnt/media on the host. I then ran each relevant service inside its container as this same UID, and punched a hole through each container's UID mapping so that specific UID doesn't get shifted like the rest of the container's range does,  it passes straight through to the mediauser's ID instead. This keeps file ownership consistent across every container that touches the media pool, regardless of which service wrote the file.


## Jellyfin

Jellyfin is set up to run on its own unprivileged LXC to keep it isolated for security and troubleshooting. It has transcoding through the intel cpu quicksync. Most media is streamed through the moonfin app for better user experience and because it connects with seer so users can request media inside the app. 

Raddarr + Sonarr

Radarr and Sonarr work pretty much the same way; they are both running in their own unprivileged LXC’s. They decide what files to grab, searching quality and size to find one that adheres to my specifications. Once qbittorent downloads the file they then import and rename it to fit jellyfins naming conventions. For my purposes with limited but not no storage 1080p in the right detail while not being too large, i also had a problem with it grabbing a non compressed version so i added a size limit, 2.5gb for a movie and 15gb for a series to aggressively grab compressed files preferably x265. 

## Prowlarr 

Prowlarr handles the indexers, which are very finicky. Rather than configuring each indexer separately inside both Radarr and Sonarr, I set it up once in Prowlarr, which connects to Radarr and Sonarr via their APIs and pushes indexer changes to both automatically.

## Qbittorent + vpn gateway

QBittorrent downloads files from multiple peers and reassembles the pieces into a complete file. To stay anonymous, all of its traffic is routed through a WireGuard VPN gateway running in its own, qBittorrent has no other way out, which creates a network level kill switch if the gateway goes down, qBittorrent goes down with it, rather than falling back to a direct connection. 

qBittorrent reaches the gateway through vmbr1, a virtual bridge both qBittorrent's eth1 and the gateway’s eth1 plug into this same bridge, giving the two containers a private link that nothing else on the LAN can see or reach.

On the gateway side, a separate interface sits on the regular LAN with normal internet access. This is what the gateway uses to perform the WireGuard handshake with the vpn’s server and establish a wg0 tunnel interface. The gateway then forwards traffic arriving on its eth1 out through wg0, and NATs it so it looks like it's coming from itself. 

Back on qBittorrent's side, its own eth0 sits on the LAN too, but deliberately has no gateway configured so even though it's reachable from the LAN, it has no route to anywhere beyond the LAN through that interface. Combined with an explicit firewall rule on the gateway that drops any eth1 traffic not destined for wg0, qBittorrent has exactly one way out to the internet, through the bridge, through the LXC, and finally through the tunnel. No fallback path if any link in that chain breaks.
 
