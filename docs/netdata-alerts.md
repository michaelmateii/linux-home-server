# Netdata Alert Coverage

The live Netdata instance exposes a much larger default alert catalog than is useful to reproduce in a portfolio repository. A configuration export contained 57 alert definitions covering system, network, disk, container, process, and memory conditions.

Rather than committing the full raw export, this repository documents the alert categories that are most relevant to the homelab.

## Alerts relevant to this server

| Alert | What it watches | Why it matters here |
|---|---|---|
| `docker_container_down` | Docker container state | Detects stopped containers in the Compose stack |
| `docker_container_unhealthy` | Docker health state | Surfaces containers that remain running but fail health checks |
| `disk_space_usage` | Filesystem space | Important because the NVMe filled during download/transcode testing |
| `disk_inode_usage` | Filesystem inode usage | Detects a different class of filesystem exhaustion |
| `nvme_device_critical_warnings_state` | NVMe critical warning state | Adds storage-health coverage for the system SSD |
| `10min_cpu_usage` | Sustained CPU usage | Useful during transcoding and other heavy work |
| `ram_in_use` | Memory utilization | Tracks pressure on the 16 GB host |
| `used_swap` | Swap consumption | Helps identify sustained memory pressure |
| `system_file_descriptors_utilization` | System file descriptors | Useful for a host running many networked services |
| `netfilter_conntrack_full` | Connection tracking capacity | Relevant to a Docker/NAT-heavy network stack |

The compact machine-readable subset is available in [`netdata-alerts-summary.csv`](netdata-alerts-summary.csv).

## Custom Docker-down rule

The most important custom rule added during the project is the generic Docker container-down template:

```text
template: docker_container_down
on: docker.container_state
chart labels: container_name=*
every: 10s
lookup: average -10s of exited
warn: $this > 0
```

It replaced an older hard-coded list of selected containers. Using `container_name=*` means newly added containers are covered automatically.

## Why the complete alert export is not published

The full Netdata alert export is intentionally excluded because it is mostly generated/default operational metadata rather than infrastructure authored for this project.

Publishing every default rule would make the repository noisier without demonstrating additional work. The repository instead keeps:

- the custom alert configuration
- a compact summary of relevant alert categories
- documentation explaining why those alerts matter
- sanitized screenshots of the monitoring dashboards

This keeps the monitoring section representative of the real server without presenting Netdata defaults as custom work.
