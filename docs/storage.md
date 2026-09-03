# Storage

## Current layout

| Purpose | Mount | Filesystem | Approx. usable size | Snapshot use |
|---|---|---|---:|---:|
| Debian, Docker data, download staging | `/` | ext4 on NVMe | 216 GiB | 58 GiB |
| Media library | `/srv/media` | ext4 (`noatime`) | 2.7 TiB | 659 GiB |

Snapshot:

```text
/srv/docker       ~2.8 GiB
/srv/downloads    ~44 GiB
/srv/media/movies ~336 GiB
/srv/media/tv     ~323 GiB
```

The media HDD is mounted persistently through `/etc/fstab`:

```fstab
UUID=<REDACTED> /srv/media ext4 defaults,noatime 0 2
```

## Hardlink lesson

Downloads currently live on the NVMe/root filesystem while media lives on a separate HDD filesystem.

Linux hardlinks cannot cross filesystem boundaries, so even with hardlinks enabled in Sonarr/Radarr, an import from `/srv/downloads` to `/srv/media` becomes a physical copy.

This explained why completed torrents could consume NVMe space while already-imported copies existed on the HDD.

## Disk troubleshooting

Useful commands during the project included:

```bash
df -h
sudo du -xhd1 /
sudo du -xhd1 /srv
sudo lsof +L1
docker system df
```

`lsof +L1` identified a deleted file that was still held open by qBittorrent, explaining disk usage that was invisible to normal directory listings.
