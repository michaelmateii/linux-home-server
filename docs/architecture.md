# Architecture

This page documents the current architecture of my personal Debian home server.

The system was built as a practical homelab for learning Linux administration, Docker Compose, networking, storage management, monitoring, self-hosting, hardware-accelerated media playback, and troubleshooting.

The diagram below represents the **actual current setup**. It intentionally separates public ingress, private administration, VPN-routed download traffic, application integrations, and physical storage.

## Architecture Diagram

```mermaid
flowchart TB

    %% =========================================================
    %% EXTERNAL ACCESS
    %% =========================================================

    RemoteClients["Remote clients<br/>Web • Mobile • TV"]
    PublicInternet(("Public Internet"))
    Router["Home Router / NAT<br/>WAN forwards:<br/>TCP 80<br/>TCP 443<br/>UDP 443"]

    LANClients["Local LAN clients<br/>TV • Phone • Mac / PC"]

    Tailscale["Tailscale<br/>Private administration<br/>Funnel disabled"]

    VPNProvider["VPN Provider<br/>OpenVPN"]

    %% =========================================================
    %% PHYSICAL SERVER
    %% =========================================================

    subgraph Host["Debian 13 (Trixie) Home Server<br/>Intel Core i5-8600K • 6C/6T • 16 GB RAM<br/>Docker Engine 29.7.2 • Docker Compose v5.5.0"]

        %% =====================================================
        %% DOCKER COMPOSE
        %% =====================================================

        subgraph Docker["Docker Compose Stack"]

            Caddy["Caddy<br/>HTTPS Reverse Proxy<br/>:80 / :443"]

            Jellyfin["Jellyfin :8096<br/>Media Server<br/>Intel Quick Sync (QSV)<br/>/dev/dri/renderD128"]

            Jellyseerr["Jellyseerr :5055<br/>Request Interface"]

            Sonarr["Sonarr :8989<br/>TV Automation"]

            Radarr["Radarr :7878<br/>Movie Automation"]

            Prowlarr["Prowlarr :9696<br/>Indexer Integration"]

            Bazarr["Bazarr :6767<br/>Subtitle Management"]

            Flaresolverr["FlareSolverr :8191<br/>Optional Support Service for<br/>Compatible Indexer Workflows"]

            Qbit["qBittorrent :8080<br/>Download Client"]

            Gluetun["Gluetun<br/>VPN Gateway<br/>OpenVPN"]

            Netdata["Netdata :19999<br/>Host / Docker / Disk Monitoring"]

            SAB["SABnzbd :8081<br/>Installed<br/>Not Currently Used"]

            %% -------------------------------------------------
            %% PUBLIC REVERSE PROXY
            %% -------------------------------------------------

            Caddy -->|HTTPS reverse proxy| Jellyfin
            Caddy -->|HTTPS reverse proxy| Jellyseerr

            %% -------------------------------------------------
            %% REQUEST MANAGEMENT
            %% -------------------------------------------------

            Jellyseerr -->|Requests| Sonarr
            Jellyseerr -->|Requests| Radarr
            Jellyseerr <-->|Library / user integration| Jellyfin

            %% -------------------------------------------------
            %% INDEXER MANAGEMENT
            %% -------------------------------------------------

            Prowlarr -->|Indexer sync| Sonarr
            Prowlarr -->|Indexer sync| Radarr
            Flaresolverr -.->|Optional support| Prowlarr

            %% -------------------------------------------------
            %% DOWNLOAD CLIENT
            %% -------------------------------------------------

            Sonarr -->|Download client| Qbit
            Radarr -->|Download client| Qbit

            %% -------------------------------------------------
            %% SUBTITLES
            %% -------------------------------------------------

            Bazarr <-->|Library metadata| Sonarr
            Bazarr <-->|Library metadata| Radarr

            %% -------------------------------------------------
            %% VPN NETWORK NAMESPACE
            %% -------------------------------------------------

            Qbit -->|"network_mode: service:gluetun"| Gluetun
        end

        %% =====================================================
        %% NVME STORAGE
        %% =====================================================

        subgraph NVMe["250 GB NVMe SSD • ext4"]

            Root["/<br/>Debian OS"]

            DockerData["/srv/docker<br/>Persistent Container Configuration"]

            Downloads["/srv/downloads<br/>complete / incomplete"]
        end

        %% =====================================================
        %% MEDIA STORAGE
        %% =====================================================

        subgraph HDD["3 TB HDD • ext4 • mounted at /srv/media"]

            Media["/srv/media"]

            Movies["/srv/media/movies"]

            TV["/srv/media/tv"]

            Media --> Movies
            Media --> TV
        end

        %% -----------------------------------------------------
        %% DOWNLOAD / IMPORT STORAGE FLOW
        %% -----------------------------------------------------

        Qbit -->|Downloads| Downloads

        Downloads -->|"Import / Copy<br/>Separate filesystems — NO HARDLINKS"| Media

        %% -----------------------------------------------------
        %% MEDIA ACCESS
        %% -----------------------------------------------------

        Media -->|Reads media| Jellyfin

        Bazarr -->|Writes external SRT subtitles| Movies
        Bazarr -->|Writes external SRT subtitles| TV

        %% -----------------------------------------------------
        %% MONITORING
        %% -----------------------------------------------------

        Netdata -.->|CPU / RAM / load / processes| Host
        Netdata -.->|Container state / resource usage| Docker
        Netdata -.->|Disk usage / I/O / SMART| NVMe
        Netdata -.->|Disk usage / I/O / SMART| HDD
    end

    %% =========================================================
    %% PUBLIC INGRESS
    %% =========================================================

    RemoteClients --> PublicInternet
    PublicInternet --> Router
    Router -->|TCP 80 / TCP 443 / UDP 443| Caddy

    %% =========================================================
    %% LOCAL ACCESS
    %% =========================================================

    LANClients -->|Direct local playback| Jellyfin

    %% =========================================================
    %% PRIVATE ADMINISTRATION
    %% =========================================================

    Tailscale -.->|SSH / private administration| Host
    Tailscale -.->|Private management UI access| Docker

    %% =========================================================
    %% VPN EGRESS
    %% =========================================================

    Gluetun --> VPNProvider
    VPNProvider --> PublicInternet

    %% =========================================================
    %% STORAGE DESIGN NOTE
    %% =========================================================

    HardlinkNote["Storage design note:<br/>/srv/downloads is on the NVMe<br/>/srv/media is on a separate HDD filesystem<br/>Linux hardlinks cannot cross these filesystems"]

    Downloads -.-> HardlinkNote
    Media -.-> HardlinkNote
```

## Host Platform

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

## Public Access

Public web access follows a deliberately small ingress path:

```text
Remote client
    ↓
Internet
    ↓
Home router / NAT
    ↓
Caddy
   ↙   ↘
Jellyfin Jellyseerr
```

The router forwards only:

```text
TCP 80
TCP 443
UDP 443
```

Caddy is the reverse proxy and the only Docker service intentionally used as the public application entry point.

The real public DNS names are omitted from this repository and replaced with example values in the published Caddy configuration.

Administrative applications such as Sonarr, Radarr, Prowlarr, Bazarr, qBittorrent, and Netdata are **not intentionally exposed through router port forwarding**.

## Private Administration

Tailscale provides a separate private administration path.

It is used for tasks such as:

- SSH access
- remote server administration
- private access to management interfaces

Tailscale Funnel was previously tested but is disabled in the current architecture.

Tailscale therefore does not form part of the public Jellyfin/Jellyseerr ingress path.

## Docker Compose Stack

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
| FlareSolverr | Support service for compatible indexer workflows |
| Netdata | Host, Docker, disk and SMART monitoring |
| SABnzbd | Installed but not currently part of the active workflow |

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

FlareSolverr is an auxiliary support service for compatible indexer workflows.

It is not part of the primary download path.

### Download flow

Sonarr and Radarr use qBittorrent as the active download client:

```text
Sonarr ─┐
        ├── qBittorrent
Radarr ─┘
```

SABnzbd is installed but currently unused, so it is not represented as part of the active download workflow.

## VPN Isolation

qBittorrent does not use its own normal Docker network namespace.

The Compose configuration uses:

```yaml
network_mode: "service:gluetun"
```

This causes qBittorrent to share Gluetun's network namespace.

The outbound path is therefore:

```text
qBittorrent
    ↓
Gluetun
    ↓
VPN Provider
    ↓
Internet
```

Gluetun currently uses OpenVPN with VPN port forwarding enabled. The real VPN provider and region are intentionally omitted from the public repository.

This VPN egress path is separate from the public Caddy ingress path.

## Jellyfin Hardware Acceleration

Jellyfin is given access to the Intel integrated GPU through:

```text
/dev/dri
```

Jellyfin is configured to use Intel Quick Sync Video with:

```text
/dev/dri/renderD128
```

This allows supported transcoding workloads to use the integrated GPU rather than relying entirely on CPU-based software transcoding.

## Storage Architecture

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

## Import Flow and Hardlink Limitation

The current storage layout creates an important limitation:

```text
/srv/downloads
     │
     │ NVMe filesystem
     ▼
 Import / Copy
     │
     │ separate filesystem
     ▼
/srv/media
     HDD
```

Linux hardlinks cannot cross filesystem boundaries.

Although hardlink support is enabled in Sonarr and Radarr, a file downloaded to the NVMe cannot be hardlinked into the HDD library.

Imports therefore become physical copies.

This became an important troubleshooting lesson because a completed download could remain on the NVMe for seeding while another full copy already existed in `/srv/media`.

A future storage redesign could place downloads and media on the same filesystem if functional hardlinks become a priority.

## Subtitle Management

Bazarr integrates with Sonarr and Radarr for library information.

It writes external subtitle files directly alongside media:

```text
Bazarr
├── /srv/media/movies
└── /srv/media/tv
```

External SRT subtitles were also useful for avoiding unnecessary playback transcoding caused by some image-based subtitle formats.

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

A custom Netdata health rule also watches for stopped Docker containers.

Raw Netdata databases, metric exports, host identifiers and runtime data are deliberately excluded from the public repository.

## Design Boundaries

This is a personal homelab rather than production infrastructure.

The architecture deliberately documents several limitations rather than hiding them:

- management services are designed primarily for trusted LAN/private access
- downloads and media currently live on separate filesystems
- hardlinks therefore cannot be used between them
- SABnzbd is installed but not currently used
- no automated backup system has been implemented yet
- Tailscale Funnel is disabled
- Caddy is the intentional public web entry point
- monitoring is provided by Netdata alongside standard Linux diagnostic tools

These limitations form part of the learning value of the project and provide clear areas for future improvement.