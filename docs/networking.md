# Networking

## Overview

The server uses three distinct networking layers:

1. **Home LAN** for normal local access.
2. **Caddy over HTTPS** as the intentional public entry point for selected services.
3. **Tailscale** for private remote administration.

A separate **Gluetun** container provides the network namespace used by qBittorrent.

## Public access

Caddy is the only service intentionally designed to receive public web traffic.

The live Compose publishes:

```text
TCP 80
TCP 443
UDP 443
```

Caddy then reverse-proxies requests internally to:

```text
Jellyfin    -> jellyfin:8096
Jellyseerr  -> jellyseerr:5055
```

The public repository does not include the real DNS hostnames.

UDP 443 is exposed alongside TCP 443 so Caddy can support HTTP/3 when available.

## Private administration

Tailscale remains installed and active for private remote administration. It is used as a private path for tasks such as SSH and management access rather than as the public media entry point.

Tailscale Funnel was tested earlier in the project but is **not part of the final design**. It was explicitly reset/disabled after Caddy-based HTTPS access was established.

No Tailscale device names, account identifiers, addresses, or tailnet domains are published in this repository.

## Docker networking

Docker Compose creates an internal bridge network for most containers.

Services such as Jellyfin, Radarr, Sonarr, Bazarr, Prowlarr, Jellyseerr, Caddy, and Netdata communicate through Docker's internal DNS/service names.

Docker translates published Compose ports into host NAT/filter rules. During troubleshooting I inspected the generated nftables/iptables-nft rules to understand how incoming traffic was DNATed to the appropriate container.

Raw firewall output is intentionally not included because it contains live container addresses and host/network details.

## qBittorrent through Gluetun

qBittorrent uses:

```yaml
network_mode: "service:gluetun"
```

This means qBittorrent shares Gluetun's network namespace instead of using its own normal Docker network stack.

Gluetun receives:

```yaml
cap_add:
  - NET_ADMIN

devices:
  - /dev/net/tun:/dev/net/tun
```

and qBittorrent's Web UI is exposed through the Gluetun container.

VPN credentials are never stored in the public Compose file. The public example references environment variables instead.

## Listener review

As part of hardening and documentation, I reviewed active listeners with:

```bash
sudo ss -lntup
```

This exposed two useful findings:

- Tailscale Funnel was still configured from earlier testing and was removed.
- `iperf3` was running as an enabled systemd service on TCP 5201 after bandwidth testing.

The iperf3 service was disabled when it was no longer needed:

```bash
sudo systemctl disable --now iperf3
```

This was a useful reminder that temporary diagnostic services can become permanent exposure if they are installed as enabled services.

## Host firewall state

The server does not currently use UFW.

Docker and Tailscale manage their own nftables/iptables-nft rules, while the host INPUT policy is not configured as a general deny-by-default firewall.

For that reason, the public example Compose uses safer bind-address defaults for management interfaces. The live server is documented as a home-LAN system behind a router/NAT boundary, not as a hardened internet-facing production host.

## Management UI exposure

On the live system, several management UIs are bound to the host LAN interfaces for convenience.

The public template is intentionally more conservative and defaults management services to:

```text
127.0.0.1
```

through the `LAN_BIND_IP` environment variable.

This is a **sanitized safer template choice**, not a claim that the live server already uses localhost-only bindings.

## Security boundary

The intended design is:

```text
Internet
   |
Router / NAT
   |
80 / 443
   |
Caddy
  |----> Jellyfin
  `----> Jellyseerr


Private administration
   |
LAN / Tailscale
   |
SSH + management UIs


qBittorrent
   |
Gluetun
   |
VPN
```

Only services deliberately selected for public use should be forwarded by the router. Router configuration, public IP addresses, private IP addresses, and real DNS names are deliberately excluded from the repository.
