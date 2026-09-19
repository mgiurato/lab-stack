# lab-stack

A low-trust sandbox for one-off experiments: a self-hosted PDF editor, a local LLM, a
tool someone linked to that looked interesting. It is the fourth stack in the lab, and
the only one that is allowed to break.

> **Not deployed yet.** This repo is complete and reviewed, but nothing in it is
> running. It is waiting on the second Proxmox node — see
> [Deploying it](#deploying-it). CT 103 is deliberately not an option; the reasoning is
> in [docs/architecture.md](docs/architecture.md#why-not-ct-103).

## What makes this one different

The other three stacks — `network-stack`, `monitoring-stack`, `documentation-stack` —
exist to provide a service, and are built so that service keeps working. This one exists
to run things whose behaviour nobody has checked. Three consequences shape the whole
repo:

| The others | This one |
| --- | --- |
| Data is backed up | Workload state is **disposable** and gitignored |
| A service may be depended on | **Nothing may depend on a workload here** |
| Egress within the LAN is unrestricted | Egress to LAN and VPN is **dropped by default** |

That third row is the substantive one, and it's what a sandbox means in practice: a
workload here can reach the internet, but it cannot reach Pi-hole's admin page, the
Proxmox API, another LXC, or a VPN client. See
[firewall/README.md](firewall/README.md#egress-containment).

## Layout

```
compose.yml                    Workloads + the two host agents + docs-server
workloads/                     One directory per experiment; state gitignored
monitoring/                    node-exporter and Alloy — this host's own agents
firewall/apply-rules.sh        Ingress allowlist + egress containment
docs-server/                   Read-only CORS markdown for docs.lan, :8092
docs/                          architecture, networking, operations, proxmox, security
systemd/                       Two units: the stack, and the firewall
```

## Ports

| Port | Service | Reachable from |
| --- | --- | --- |
| 8100 | `stirling-pdf` (example workload) | LAN + VPN |
| 8092 | `docs-server` | LAN + VPN |
| 9100 | `node-exporter` | `MONITORING_HOST` only |
| 12345 | `alloy` | `MONITORING_HOST` only |

Workload ports are allocated from **8100 upwards**, one per experiment, so they never
collide with the 8090–8092 docs servers or with anything the other stacks publish.

## Deploying it

When the new node exists and the LXC is created:

1. `git clone` this repo to `/root/lab-stack`.
2. `cp .env.example .env` and fill it in — in particular set `LAB_HOST` to the new
   container's real address, which the placeholder in `.env.example` is not.
3. Generate the scrape credential:
   `cp monitoring/web-config/node-exporter.yml.example monitoring/web-config/node-exporter.yml`,
   replace the hash, and put the matching plaintext in monitoring-stack's
   `prometheus/secrets/lab-stack-scrape-password` (owned `65534:65534`, mode `400`).
4. `cp systemd/*.service /etc/systemd/system/ && systemctl enable --now lab-stack lab-stack-firewall`
5. On the **other** stacks, four small edits — all listed with their exact file and
   line in [docs/operations.md](docs/operations.md#joining-the-lab-to-the-rest-of-the-lab).

Then verify with the checks in
[docs/operations.md](docs/operations.md#verifying-a-deployment) — including the egress
test, which is the one that is easy to assume and wrong.

## Adding a workload

The full recipe is in [workloads/README.md](workloads/README.md). The short version:
give it a directory, publish it on the next free port from 8100, add that port to
`DOCKER_PORTS` in `firewall/apply-rules.sh`, enable whatever login it has, and pin its
image tag. If it needs to reach something on the LAN, that is a deliberate exception in
the egress rules — not a reason to remove them.
