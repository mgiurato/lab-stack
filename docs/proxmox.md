# Proxmox — lab-stack

## Container

Not yet created. Intended placement is the second node in the cluster, not the minipc
that hosts CT 103 — see [architecture.md](architecture.md#why-not-ct-103).

| Setting | Value | Why |
| --- | --- | --- |
| Type | LXC, **unprivileged** | Same as every other container in the lab. Non-negotiable here: unvetted code runs in it |
| Nesting | `nesting=1` | Required to run Docker inside an LXC |
| RAM | 4 GiB to start, 8 GiB with SnapOtter | Enough for several small workloads. SnapOtter's own limits (app 6 GB, PostgreSQL 1 GB, Redis 512 MB) add up to 7.5 GiB; a local LLM wants considerably more — size for what you intend to run |
| Cores | 2+ | CT 103's single core is a real constraint; don't repeat it |
| Disk | 32 GiB+ | Images dominate. Model weights do not fit in a modest disk |

Record the real values here once the container exists, and update the address in
`.env`.

## Backups

**Workload data is deliberately not backed up.** `workloads/*/data/` and
`workloads/*/config/` are disposable by design — see
[architecture.md](architecture.md#promotion). If something in there would be missed, the
workload has outgrown this stack.

What is worth a `vzdump` of the container: `.env` and
`monitoring/web-config/node-exporter.yml`, both gitignored and both annoying rather than
catastrophic to lose. Everything else is in git.

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
