# Security — lab-stack

## Trust model

The other three stacks treat the LAN as the trust boundary. This one does not trust
itself.

A workload here is assumed to be, at best, unreviewed: an image whose provenance nobody
checked, running code whose behaviour nobody characterised. The stack is built so that
assumption is survivable rather than being talked out of.

| Direction | Posture |
| --- | --- |
| LAN → workload | Allowed. Each workload carries its own login; Caddy adds `basic_auth` on the browser route |
| LAN → agents | `MONITORING_HOST` only, via `PEER_PORTS` |
| Workload → LAN/VPN | **Dropped**, except DNS to `NETWORK_HOST` and Loki to `MONITORING_HOST` |
| Workload → internet | Allowed — accepted risk, see below |

## What the egress rules do and do not stop

They stop a workload reaching another host on the LAN or a VPN client. Concretely, they
stop the scenarios that make an untrusted container on a flat home network dangerous:
scanning `192.168.1.0/24`, reaching Pi-hole's admin page or the Proxmox API on `:8006`,
and pivoting to a NAS or another LXC.

They do **not** stop:

- **Anything the workload does to its own LXC.** Containment is at the network layer.
  The container boundary is what protects the host, which is why every service sets
  `no-new-privileges`, nothing runs `privileged`, and the LXC is unprivileged.
- **Outbound internet access.** A malicious image can exfiltrate whatever it can read
  and fetch a second stage. This is a deliberate trade: a playground with no internet
  cannot pull a model or a dependency, and would not get used. The mitigation is that
  what it can read is limited to its own disposable volumes.
- **Anything reaching it from the LAN.** Any device on the LAN can open a workload's
  published port. Hence the per-workload login requirement below.

## Rules

**Every workload with an authentication option must have it enabled.** Caddy's
`basic_auth` gates the browser route through `*.lan` only — the published port on
`LAB_HOST` is directly reachable from any LAN device, and a reverse proxy in front of an
origin that is still reachable on its own port is not an access control.

**Never grant this stack an entry in another stack's `PEER_HOSTS`.** The lab initiates
nothing towards the other stacks except Alloy's log push. An entry in
`monitoring-stack`'s `PEER_HOSTS` would let a compromised workload read every metric in
the lab and silence every alert.

**`alloy` is the highest-value target here.** It mounts `docker.sock` read-only, which
is enough to enumerate every container and read its logs, and it has no authentication
of its own — the firewall allowlist is the only control on `:12345`. It is a per-host
agent and cannot be centralised, so the mitigations are the allowlist and the pinned
image, both of which matter more here than on the other stacks.

**Workload state is not backed up**, so it must never hold anything that matters. This
is a security property as much as an operational one: it bounds what a compromised
workload has access to.

## Secrets

| Secret | Where | Notes |
| --- | --- | --- |
| Workload logins | `.env` | Gitignored. Seeded at first start; change in the UI afterwards |
| Scrape credential hash | `monitoring/web-config/node-exporter.yml` | Gitignored; `.example` is committed. Plaintext lives in monitoring-stack's `prometheus/secrets/lab-stack-scrape-password` |

`docs-server` mounts markdown file by file, never the repo root — mounting the root
would serve `.env`. `docs/audits/` is excluded from it too.

## Not verifiable from inside this container

- Router-level port forwarding. Nothing in this stack is intended to be
  internet-reachable; that depends on the router not forwarding to `LAB_HOST`, which
  cannot be checked from here.
- Proxmox host-level settings (LXC privilege config, resource limits).
- Whether the router advertises IPv6. The `ip6tables` mirror in
  `firewall/apply-rules.sh` means the published ports fail closed either way.
