# Networking — lab-stack

## Addresses

| Name | Value | Set in |
| --- | --- | --- |
| `LAB_HOST` | `192.168.1.50` | `.env` (the same value is in `.env.example`) |
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
| 8102 | `cotta-app-1` (cotta, not this stack) | LAN + VPN | `DOCKER_PORTS` |

Workload ports start at **8100** and count upwards; 8102 is taken by cotta, which shares
this LXC but is its own project (see
[architecture.md](architecture.md#cotta-the-other-tenant-of-ct-104)), so the next free
one is 8103. The 8090–8092 range is the lab's docs-servers, each on its own host:
network-stack 8090 and proxmox-hosts 8092 on `192.168.1.39`, monitoring-stack 8091,
media-stack 8090, health-stack 8092, and this stack 8092.

## Flows

Inbound:

- A LAN or VPN browser → `search.lan` / `search.pelae-lan.duckdns.org` or `files.lan` /
  `files.pelae-lan.duckdns.org` → network-stack's Caddy → `LAB_HOST:8100` or
  `LAB_HOST:8101`. The DuckDNS names resolve publicly to network-stack, so they work
  whatever resolver the client uses; the `.lan` names need Pi-hole. Caddy adds
  `basic_auth` on the SearXNG route only — SearXNG has no login of its own, while
  SnapOtter has one and sends its own `Authorization` header, which a proxy-level login
  would collide with.
- A LAN browser → `LAB_HOST:8100` or `:8101` directly. This bypasses Caddy, which is
  why each workload that has a login needs its own on.
- `docs.lan` → documentation-stack's Caddy → `LAB_HOST:8092`, reverse-proxied
  server-side under `/peer-docs/lab-stack/`.
- monitoring-stack's Prometheus → `LAB_HOST:9100` and `:12345`.

Outbound:

- `alloy` → `MONITORING_HOST:3100`. The only flow this stack initiates towards another.
- Any container → the internet. Allowed.
- Any container → anywhere else on `192.168.1.0/24` or `10.8.0.0/24`. **Dropped.**

Cotta, the other project on this LXC, has its port in the table above but is otherwise
outside these flows:

- A LAN or VPN browser or MCP client → `cotta.lan` or `cotta.pelae-lan.duckdns.org` →
  network-stack's Caddy → `LAB_HOST:8102`. The app answers only those two host names.
- Cotta's containers sit on `cotta_default`, not `172.22.0.0/16`, so the egress drops
  above do not apply to them.

## DNS

CT 104 does not use Pi-hole. Its LXC config sets no `nameserver`, so it inherits pve-01's
`/etc/resolv.conf`: `nameserver 192.168.1.1`, the router, with `search lan`. The host
and every container therefore resolve public names through the router, and `*.lan` names
do not resolve here at all (`getent hosts docs.lan` fails) — use a LAN address from
`.env`, as the cross-stack rule says anyway, or a `*.pelae-lan.duckdns.org` name.

Containers use Docker's embedded resolver, which forwards to the host's `resolv.conf`
and so to the router. That forwarding happens as host traffic, not container traffic, so
it is not subject to the egress rules — and since it never goes to `NETWORK_HOST`, the
`udp/53` and `tcp/53` exceptions to `NETWORK_HOST` are not what lets ordinary lookups
work. They exist for the workload that sets Pi-hole as its own resolver or digs it
directly — without them that fails in a way that looks like broken DNS rather than a
firewall drop.

## Adding a `*.lan` hostname

The record and the site blocks live in `network-stack`, not here:

1. `dns/pihole/etc-dnsmasq.d/lan.conf` — point the hostname at network-stack's address
   (Caddy proxies onward; the record does not point at `LAB_HOST`).
2. `proxy/caddy/Caddyfile` — an `http://<name>.lan` site block with
   `reverse_proxy {$LAB_HOST}:<port>`, and an `https://<name>.lan` twin that adds
   `tls internal`. Add a `basic_auth` block to both **only if the workload has no login
   of its own** (SearXNG). A workload with a login keeps it as its only one, and one that
   sends its own `Authorization` header (SnapOtter) must not get `basic_auth` — the two
   collide.
3. In the same Caddyfile, inside the `*.{$DUCKDNS_LAN_DOMAIN}.duckdns.org` site block, a
   matching handler — `@<name> host <name>.{$DUCKDNS_LAN_DOMAIN}.duckdns.org` and
   `handle @<name> { reverse_proxy {$LAB_HOST}:<port> }`, with the same `basic_auth`
   decision as step 2. The wildcard DuckDNS record and certificate already cover the
   name; this is the entry point that works with any client resolver.
4. `documentation-stack`'s `caddy/site/index.html` — a card on the landing page.
5. `docker compose restart pihole` and `docker compose up -d --force-recreate caddy`.

Steps 2 and 3 need `LAB_HOST` in network-stack's `.env` and in its `caddy` service's
`environment:` block.
