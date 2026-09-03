# Architecture

This page documents the current architecture of my personal Debian home server.

The system was built as a practical homelab for learning Linux administration, Docker Compose, networking, storage management, monitoring, self-hosting, hardware-accelerated media playback, and troubleshooting.

The documentation below represents the **actual current setup** while omitting real IP addresses, DNS names, credentials, account identifiers, and runtime data.

## Host platform

The current server is built from standard desktop hardware rather than dedicated server hardware:

| Component | Current configuration |
|---|---|
| Operating system | Debian GNU/Linux 13 (Trixie) |
| CPU | Intel Core i5-8600K |
| CPU layout | 6 cores / 6 threads |
| RAM | 16 GB |
| System disk | 250 GB NVMe SSD |
| Media disk | 3 TB SATA HDD |
| Container runtime | Docker Engine 29.7.2 |
| Orchestration | Docker Compose v5.5.0 |

The server runs headless and is normally administered remotely.

## Architecture summary

The server separates public ingress, private administration, VPN-routed download traffic, application integrations, monitoring, and physical storage.

### Public web access

```text
Remote clients
      |
   Internet
      |
Home router / NAT
      |
 TCP 80 / TCP 443 / UDP 443
      |
    Caddy
   /     \
Jellyfin  Jellyseerr
```

Caddy is the only Docker service intentionally used as the public application entry point.

The real public DNS names are omitted from the repository and replaced with example values in the published Caddy configuration.

Administrative applications such as Sonarr, Radarr, Prowlarr, Bazarr, qBittorrent, and Netdata are not intentionally exposed through router port forwarding.

### Private administration

Tailscale provides a separate private administration path for:

- SSH access
- remote server administration
- private access to management interfaces

Tailscale Funnel was previously tested but is disabled in the current architecture. It is not part of the public Jellyfin/Jellyseerr ingress path.

### VPN-routed download traffic

qBittorrent shares Gluetun's network namespace:

```yaml
network_mode: "service:gluetun"
```

The outbound path is therefore:

```text
qBittorrent
    |
  Gluetun
    |
VPN provider
    |
 Internet
```

The public documentation does not include real VPN credentials or provider-specific account information.

## Docker Compose stack

The server currently runs the following services:

| Service | Purpose |
|---|---|
| Caddy | HTTPS reverse proxy |
| Jellyfin | Media server |
| Jellyseerr | Request interface |
| Sonarr | TV library automation |
| Radarr | Movie library automation |
| Prowlarr | Indexer integration |
| Bazarr | Subtitle management |
| qBittorrent | Download client |
| Gluetun | VPN gateway for qBittorrent |
| FlareSolverr | Optional support service for compatible indexer workflows |
| Netdata | Host, Docker, disk and SMART monitoring |
| SABnzbd | Installed but not currently part of the active workflow |

## Service relationships

### Request flow

Jellyseerr sends requests to the appropriate automation service:

```text
Jellyseerr
├── Sonarr
└── Radarr
```

Jellyseerr also integrates with Jellyfin for library and user information.

### Indexer integration

Prowlarr manages indexer integration for Sonarr and Radarr:

```text
Prowlarr
├── Sonarr
└── Radarr
```

FlareSolverr is an optional auxiliary support service for compatible indexer workflows. It is not part of the primary download path.

### Download flow

Sonarr and Radarr use qBittorrent as the active download client:

```text
Sonarr ─┐
        ├── qBittorrent
Radarr ─┘
```

qBittorrent then routes its network traffic through Gluetun.

SABnzbd is installed but currently unused, so it is not part of the active download workflow.

### Subtitle flow

Bazarr integrates with Sonarr and Radarr for library information and writes external subtitle files alongside media:

```text
Bazarr
├── /srv/media/movies
└── /srv/media/tv
```

External SRT subtitles were also useful for avoiding unnecessary playback transcoding caused by some image-based subtitle formats.

## Jellyfin hardware acceleration

Jellyfin is given access to the Intel integrated GPU through:

```text
/dev/dri
```

Jellyfin is configured to use Intel Quick Sync Video with:

```text
/dev/dri/renderD128
```

This allows supported transcoding workloads to use the integrated GPU rather than relying entirely on CPU-based software transcoding.

![Jellyfin Intel Quick Sync configuration](../screenshots/jellyfin-qsv-transcoding.png)

## Storage architecture

The server currently uses two separate physical filesystems.

### NVMe system disk

```text
250 GB NVMe SSD
ext4

/
├── Debian
├── /srv/docker
└── /srv/downloads
    ├── complete
    └── incomplete
```

`/srv/docker` contains persistent application configuration.

`/srv/downloads` is the staging area used by the download client.

### Media HDD

```text
3 TB HDD
ext4
mounted at /srv/media

/srv/media
├── movies
└── tv
```

The HDD contains the final Jellyfin media libraries.

## Import flow and hardlink limitation

The current storage layout creates an important limitation:

```text
/srv/downloads
      |
      | NVMe filesystem
      v
 Import / Copy
      |
      | separate filesystem
      v
/srv/media
    HDD
```

Linux hardlinks cannot cross filesystem boundaries.

Although hardlink support is enabled in Sonarr and Radarr, a file downloaded to the NVMe cannot be hardlinked into the HDD library.

Imports therefore become physical copies.

This became an important troubleshooting lesson because a completed download could remain on the NVMe for seeding while another full copy already existed in `/srv/media`.

A future storage redesign could place downloads and media on the same filesystem if functional hardlinks become a priority.

## Monitoring

Netdata runs as a Docker container but receives enough host access to monitor the underlying Debian system.

It is used for:

- CPU usage and system load
- memory and swap
- Docker container state
- per-container CPU and RAM use
- filesystem usage
- mount-point capacity
- disk I/O
- SMART information
- NVMe health

A custom Netdata health rule watches for stopped Docker containers.

### Monitoring screenshots

![Netdata system overview](../screenshots/netdata-system-overview.png)

![Netdata storage overview](../screenshots/netdata-storage.png)

![Netdata container overview](../screenshots/netdata-containers.png)

Raw Netdata databases, metric exports, host identifiers, and runtime data are deliberately excluded from the public repository.

## Current design boundaries

This is a personal homelab rather than production infrastructure.

The current design intentionally documents its limitations:

- management services are designed primarily for trusted LAN/private access
- downloads and media currently live on separate filesystems
- hardlinks therefore cannot be used between them
- SABnzbd is installed but not currently used
- no automated backup system has been implemented yet
- Tailscale Funnel is disabled
- Caddy is the intentional public web entry point
- monitoring is provided by Netdata alongside standard Linux diagnostic tools

These limitations form part of the learning value of the project and provide clear areas for future improvement.
