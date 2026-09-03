# Troubleshooting Notes

## Deleted file still consuming space

The root filesystem stayed unexpectedly full after deleting a large file.

```bash
df -h
du -xhd1 /
sudo lsof +L1
```

`lsof +L1` showed that qBittorrent still held the deleted file open. Restarting the container released the file descriptor and returned the disk space.

## Jellyfin transcode cache growth

The Jellyfin transcode cache grew to tens of gigabytes during playback testing. `du` isolated the directory and stale transcode data was cleared after stopping the container.

## Subtitle-triggered transcoding

Image-based PGS subtitles caused some clients to require video transcoding and produced unreliable playback.

The final approach was:

- Bazarr for external SRT subtitles
- Jellyfin configured to allow text subtitles while ignoring image-based embedded subtitles

## Hardlinks and filesystems

A consistent `/data` path inside containers did not solve the fact that hardlinks cannot cross host filesystem boundaries.

## Networking cleanup

Reviewing `ss -lntup` found:
- an old Tailscale Funnel configuration left from testing
- an iperf3 server running as an enabled systemd service

Both were removed from the active design once no longer needed.
