# Networking — lab-stack

## Addresses

| Name | Value | Set in |
| --- | --- | --- |
| `LAB_HOST` | this LXC's LAN address | `.env` — **the `.env.example` value is a placeholder** |
| `NETWORK_HOST` | `192.168.1.39` | `.env` |
| `MONITORING_HOST` | `192.168.1.70` | `.env` |
| Bridge subnet | `172.22.0.0/16` | `compose.yml` `ipam`, and matched in `firewall/apply-rules.sh` |

Within this project, containers reach each other by container name. Across projects,
always a LAN address from `.env` — container-name resolution only works while stacks
share a host, and this one never will.

The bridge subnet is pinned rather than left to Docker because the egress rules match on
it. `172.19` is network-stack, `172.20` monitoring, `172.21` documentation, `172.22`
here.

## Ports

| Port | Service | Source allowed | Enforced by |
| --- | --- | --- | --- |
| 8100 | `searxng` | LAN + VPN | `DOCKER_PORTS` |
| 8101 | `snapotter` | LAN + VPN | `DOCKER_PORTS` |
| 8092 | `docs-server` | LAN + VPN | `DOCKER_PORTS` |
| 9100 | `node-exporter` | `MONITORING_HOST` | `PEER_PORTS` |
| 12345 | `alloy` | `MONITORING_HOST` | `PEER_PORTS` |

Workload ports start at **8100** and count upwards. The 8090–8092 range is taken by the
three docs-servers (network 8090, monitoring 8091, lab 8092; media-stack uses 8090 on
its own host).

## Flows

Inbound:

- A LAN browser → `search.lan` or `files.lan` → network-stack's Caddy → `LAB_HOST:8100`
  or `LAB_HOST:8101`.
- A LAN browser → `LAB_HOST:8100` or `:8101` directly. This bypasses Caddy and its
  `basic_auth`, which is why each workload that has a login needs its own on.
- `docs.lan` → the browser fetches `LAB_HOST:8092` client-side over CORS.
- monitoring-stack's Prometheus → `LAB_HOST:9100` and `:12345`.

Outbound:

- `alloy` → `MONITORING_HOST:3100`. The only flow this stack initiates towards another.
- Any container → the internet. Allowed.
- Any container → anywhere else on `192.168.1.0/24` or `10.8.0.0/24`. **Dropped.**

## DNS

Containers use Docker's embedded resolver, which forwards to the host's `resolv.conf`
and so to Pi-hole. That forwarding happens as host traffic, not container traffic, so it
is not subject to the egress rules. The explicit `udp/53` and `tcp/53` exceptions to
`NETWORK_HOST` exist for the workload that sets its own resolver or digs directly —
without them that fails in a way that looks like broken DNS rather than a firewall drop.

## Adding a `*.lan` hostname

The record and the site block live in `network-stack`, not here:

1. `dns/pihole/etc-dnsmasq.d/lan.conf` — point the hostname at network-stack's address
   (Caddy proxies onward; the record does not point at `LAB_HOST`).
2. `proxy/caddy/Caddyfile` — a site block with `reverse_proxy {$LAB_HOST}:<port>` and a
   `basic_auth` block.
3. `documentation-stack`'s `caddy/site/index.html` — a card on the landing page.
4. `docker compose restart pihole` and `docker compose up -d --force-recreate caddy`.

Step 2 needs `LAB_HOST` in network-stack's `.env` and in its `caddy` service's
`environment:` block.
