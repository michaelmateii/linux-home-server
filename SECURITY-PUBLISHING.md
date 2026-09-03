# Publication Safety Checklist

Before every public push:

- [ ] No real `.env` is staged.
- [ ] No VPN credentials, API keys, tokens, cookies, or passwords are present.
- [ ] No application credentials from Jellyfin, Sonarr, Radarr, Prowlarr, Bazarr, Jellyseerr, qBittorrent, or Netdata are present.
- [ ] No real public IP, LAN IP, Tailscale IP, DuckDNS hostname, email, machine ID, boot ID, or hardware serial is present.
- [ ] No raw application databases, logs, backups, or `/srv/docker` runtime data are included.
- [ ] No torrent files, magnet links, tracker URLs, indexer credentials, or media acquisition instructions are included.
- [ ] Screenshots are checked for usernames, IPs, hostnames, tokens, history, and media filenames.
- [ ] `git diff --cached` has been reviewed before the first public push.

Useful local checks:

```bash
git status
git diff --cached

grep -RniE   'OPENVPN_PASSWORD|OPENVPN_USER|api[_-]?key|token|password|100\.[0-9]+\.[0-9]+\.[0-9]+|192\.168\.'   . --exclude-dir=.git
```

If a real secret is ever committed, removing it in a later commit is not enough. Rotate the secret and remove it from Git history before publishing.
