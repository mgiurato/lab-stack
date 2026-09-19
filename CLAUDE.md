# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in
this repository.

## What this is

A low-trust sandbox for one-off experiments, deployed as a Docker Compose project
(`compose.yml`) in its own unprivileged Proxmox LXC. There is no application code to
build — this repo is Compose service definitions, container configs, and documentation.

**Nothing here is running yet.** The repo is complete and waiting on a Proxmox node with
capacity. Do not describe it as deployed, and do not add it to another stack's live
configuration until it is. CT 103 is not a candidate: it has 2 GiB and one vCPU, and
`docs/architecture.md` explains why colocating unvetted workloads with DNS and the VPN
is the wrong trade even if it fit.

**This is the fourth stack in the lab**, each its own repo:

| Repo | Owns |
| --- | --- |
| `network-stack` | DNS, reverse proxy, VPN, DDNS, certificates, firewall, local agents |
| `monitoring-stack` | Prometheus, Alertmanager, Grafana, Loki, Uptime Kuma |
| `documentation-stack` | `home.lan` and `docs.lan` |
| `lab-stack` | This one: disposable experiments |

## Commands

```bash
docker compose up -d                          # start/apply the stack
docker compose ps                             # status
docker compose logs -f [--tail=100] <service> # follow logs
./firewall/apply-rules.sh                     # re-apply rules (idempotent)
```

Function checks, since `Up` is not proof of health:

```bash
curl -s localhost:8092/README.md | head -1               # docs-server
curl -su prometheus:<pw> localhost:9100/metrics | head -1 # node-exporter (401 without auth)
docker exec alloy sh -c 'wget -qO- localhost:12345/-/ready'
```

No test suite, linter, or build step exists in this repo.

## The rules that make this stack different

Everything below is what separates a sandbox from just another stack. Weakening any of
them turns this into a fourth production stack with worse hygiene than the other three.

**Nothing outside this repo may depend on a workload here.** No other stack's config, no
alert, no Uptime Kuma monitor that pages. A workload that something has come to rely on
has outgrown the lab — promote it to its own stack rather than hardening it in place.

**Workload state is disposable.** `workloads/*/data/` and `workloads/*/config/` are
gitignored and not backed up. Don't add a workload's volume to a backup job; if data
there matters, that's the promotion signal above.

**Egress to the LAN and the VPN is dropped by default.** `firewall/apply-rules.sh`
allows only DNS to `NETWORK_HOST`, Loki to `MONITORING_HOST`, and established return
traffic. The internet is open — that's the point of a playground. Adding an exception is
a deliberate, commented edit; removing the catch-all drops is never the fix.

**The egress rules match `-s 172.22.0.0/16`**, the `lab-stack` bridge subnet pinned in
`compose.yml`'s `ipam` block. Those two numbers are one fact in two files. Change one and
the containment silently stops applying — it will not error, it will just stop working.

**This stack is a leaf.** It is scraped by monitoring-stack and reverse-proxied by
network-stack; it initiates nothing towards either except Alloy's log push. **Never add
`LAB_HOST` to another stack's `PEER_HOSTS`** — that would grant a sandbox the right to
read every metric in the lab.

**Agent ports stay in `PEER_PORTS`.** `node-exporter` (9100) and `alloy` (12345) have
almost no authentication — node-exporter has basic auth, Alloy has none — and Alloy
mounts `docker.sock`. Never move either into `DOCKER_PORTS`.

**Every workload that has a login must have it enabled.** Caddy's `basic_auth` on
network-stack gates the *browser* route only; the published port is reachable directly
from the LAN, so proxy-level auth is not the workload's auth.

## Conventions shared with the other three repos

- **Every image is pinned** to a specific version — never `latest`, never a bare major.
  Bump the tag deliberately.
- **Across stacks, always use a LAN address from `.env`** (`NETWORK_HOST`,
  `MONITORING_HOST`), never a container name. Container-name resolution only works while
  stacks share a host.
- **No dates** stamped on features or fixes in `README.md`/`docs/*.md` — describe the
  current state plainly and let `git log` answer "when".
- **Open issues live only in `docs/audits/infosec-audit-repo.md`**, never in a
  `README.md` or `docs/*.md`. Closed findings are deleted, not annotated — git history is
  the record. **In this repo that file is gitignored**, because the remote is public and
  a list of what is currently weak and unfixed is the one thing here that should not be:
  keep writing to it, don't commit it, and don't "fix" the ignore rule.
- **`docs-server` mounts markdown file by file, never the repo root** — that would serve
  `.env`. `docs/audits/` is deliberately excluded.
- **Docker creates a missing bind-mount source file as a *directory*.** If a container
  fails to start with a mount error, look for that first:
  `find . -type d -name '*.md' -empty -delete`.
- **Secrets are gitignored and never re-tracked.** When adding an ignore rule for
  something already committed, pair it with `git rm --cached`.

## Environment

`.env` (copy from `.env.example`) is required — `MONITORING_HOST`, `STIRLING_USERNAME`
and `STIRLING_PASSWORD` are declared required in `compose.yml` and Compose refuses to
start those services without them. `LAB_HOST` in `.env.example` is a placeholder, not a
real address; set it when the LXC exists.
