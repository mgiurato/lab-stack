# Proxmox — lab-stack

## Container

CT 104 on pve-01 (`192.168.1.50`, static, MAC `BC:24:11:50:00:04`), not on the node that
hosts CT 103 — see [architecture.md](architecture.md#why-not-ct-103). The node's own
setup is in the proxmox-hosts repository.

| Setting | Value | Why |
| --- | --- | --- |
| Type | LXC, **unprivileged** | Same as every other container in the lab. Non-negotiable here: unvetted code runs in it |
| Nesting | `nesting=1` | Required to run Docker inside an LXC |
| RAM | 8 GiB, swap 1 GiB | Room for SearXNG and SnapOtter at upstream's limits in [compose.yml](../compose.yml) (app 6 GB, PostgreSQL 1 GB, Redis 512 MB, 7.5 GiB together). A local LLM wants considerably more — size for what you intend to run |
| Cores | 2 | Pins the container to 2 of pve-01's 8 threads, so a runaway workload cannot starve Home Assistant or Jellyfin. SnapOtter's CPU limit in [compose.yml](../compose.yml) matches it: Docker refuses a limit above the cores it can see |
| Disk | 28 GB | Images dominate, and SnapOtter keeps its AI models in a volume. Model weights do not fit in a modest disk |

## Backups

**Workload data is deliberately not backed up.** `workloads/*/data/` and
`workloads/*/config/` are disposable by design -- see
[architecture.md](architecture.md#promotion). If something in there would be missed, the
workload has outgrown this stack.

The container itself is dumped nightly to the NAS (`backup-lab-stack` on pve-01: 3 daily,
2 weekly, 1 monthly; ~6 GB per dump, no local copy). That keeps `.env`,
`monitoring/web-config/node-exporter.yml` and the named volumes -- annoying rather than
catastrophic to lose -- and everything else is in git.

This is the one stack where "restore from scratch" is an acceptable recovery plan, and
the one where it is worth actually exercising — a clean rebuild is cheap here and
confirms the clone-and-configure path in the README still works.

## Host-side checklist

Not verifiable from inside the container:

- The LXC is genuinely unprivileged.
- The router does not forward any port to `LAB_HOST`. Nothing here is meant to be
  internet-reachable.
- Resource limits are set, so a runaway workload cannot starve the node. This matters
  more here than for the other stacks: an experiment allocating without bound is a
  normal Tuesday, not an incident.
