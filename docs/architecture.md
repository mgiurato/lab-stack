# Architecture — lab-stack

## What it is for

Somewhere to run things nobody has vetted. A self-hosted PDF editor, a local LLM, a
tool from a link — the category is "I want to try this", and the defining property is
that trying it must not be able to damage anything else.

Everything else in the lab is built so a service keeps working. This stack is built so
that when something in it misbehaves, the blast radius stops at its own LXC.

```
                          LAN 192.168.1.0/24
                                  │
        ┌─────────────────────────┼──────────────────────────┐
        │                         │                          │
   network-stack             monitoring-stack            lab-stack
   Caddy ──── reverse-proxies ───────────────────────────▶ workloads
                  Prometheus ── scrapes ─────────────────▶ agents
                       Loki ◀──── Alloy pushes ───────────┘
                                                          │
                              egress to LAN/VPN: DROPPED ─┘
                              egress to internet: allowed
```

The arrows are all inbound except one. That asymmetry is the design: the lab is reached,
it does not reach. The single exception is Alloy's log push, because a sandbox nobody can
see the logs of is worse than no sandbox.

## Why not CT 103

CT 103 runs `network-stack` on 2 GiB of RAM and a single vCPU (`monitoring-stack` and
`documentation-stack` have their own containers, CT 106 and CT 107). The headroom is
not enough for a local LLM by an order of magnitude, and the one core is the harder
limit.

The capacity argument is the weaker one, though, and it would go away with a bigger
container. The real reason is the trust model. CT 103 terminates the VPN and answers
every DNS query on the LAN — the two services the rest of the house notices within
seconds of them stopping. A playground is where unvetted code runs by definition.
Putting the two in one failure domain, sharing one kernel and one core, inverts the
boundary the rest of the lab is built around. `network-stack`'s own
`docs/architecture.md` already warns against exactly this in the case of Prometheus and
Loki, which are at least trusted software.

So: its own LXC on the second Proxmox node, unprivileged like the others.

## Why egress containment, and only here

The other three stacks talk to each other across the LAN constantly — Caddy proxies to
all of them, Prometheus scrapes all of them, Alloy pushes from all of them. Restricting
their egress would mean maintaining an allowlist that changes every time a service is
added.

The lab has no such traffic. It is a leaf: reached by Caddy and Prometheus, pushing only
to Loki. That makes a default-deny egress policy nearly free to maintain — three
exceptions, listed in `firewall/apply-rules.sh`, that have no reason to grow. A control
that costs nothing to keep correct is one that will still be correct in a year.

What it buys: a workload with a known CVE, or one that simply does more than its README
claims, cannot port-scan the LAN, cannot reach Pi-hole's admin interface or the Proxmox
API on `:8006`, and cannot talk to a VPN client. It can still reach the internet, which
is both the point and the accepted risk — see `docs/security.md`.

## Promotion

A workload that stops being disposable should leave. The signals, in rough order of how
often they show up:

- Something outside the lab depends on it being up.
- Its data would be missed if the LXC were deleted.
- It needs a LAN egress exception that isn't obviously temporary.

The move is the same recipe every stack here already follows: its own repo, its own
firewall script, its own docs-server, its own agents, an entry in Prometheus and a site
block in Caddy. Hardening a workload in place instead is how a sandbox quietly turns
into a fourth production stack with the weakest hygiene of the four.

## Cotta, the other tenant of CT 104

CT 104 also runs **cotta**, a homebrewing app (recipes, inventory, batches, and an MCP
endpoint), which is not part of this stack. It is a separate Compose project (`cotta`) from
its own repository, `mgiurato/homebrew-app`, checked out at `/root/homebrew-app` and
started from `/root/homebrew-app/deploy/compose.yaml` with the `tunnel` profile. Nothing
in this repo defines, starts or configures it — `docker compose` here does not see its
containers, and its settings live in `/root/homebrew-app/deploy/.env`.

| Container | Image | What it does |
| --- | --- | --- |
| `cotta-app-1` | `cotta:<commit>`, built locally from that repo | The app, published on `LAB_HOST:8102` → container port 8000, for LAN and VPN clients (including MCP) |
| `cotta-cloudflared-1` | `cloudflare/cloudflared` (pinned) | A named Cloudflare Tunnel: an outbound connection that publishes the app on a public hostname, so people outside the LAN can use it |

Its data is in the named volume `cotta_cotta-data`, and the app writes its own backups to
`/root/homebrew-app/deploy/backups`. Both are kept: that data is wanted.

It sits here rather than in its own LXC by choice, and that makes it the exception to most
of what this page says:

- **It is not disposable.** It is exactly what [Promotion](#promotion) moves out of the
  lab. It already has its own repo; it shares only the container.
- **The lab's firewall rules do not apply to it.** `firewall/apply-rules.sh` matches this
  stack's ports and the `lab-stack` subnet `172.22.0.0/16`. Cotta's project network
  (`cotta_default`) has an address range Docker picked itself, so egress containment does
  not cover it, and port 8102 has no entry in `DOCKER_PORTS`.
- **It is reachable from the internet.** The tunnel is outbound-only, so no router port
  forward is involved, but its public hostname brings traffic into CT 104, the container
  that runs unvetted lab workloads.
- **This host's agents cover it anyway.** Alloy's Docker discovery picks up every
  container on the host, so cotta's logs reach Loki with the rest, and node-exporter's
  metrics include its load.
