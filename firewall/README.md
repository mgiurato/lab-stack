# Firewall — lab-stack

`apply-rules.sh` is idempotent and runs at boot via `lab-stack-firewall.service`. It
appends to the same `DOCKER-USER` chain the other stacks use, matching only this stack's
ports and subnet, so the four scripts coexist without touching each other's rules.

It does two things. The first is the same as every other stack. The second is why this
one exists.

## Ingress

| Port | Service | Allowed from | Array |
| --- | --- | --- | --- |
| 8092 | `docs-server` | LAN + VPN | `DOCKER_PORTS` |
| 8100 | `stirling-pdf` | LAN + VPN | `DOCKER_PORTS` |
| 9100 | `node-exporter` | `MONITORING_HOST` | `PEER_PORTS` |
| 12345 | `alloy` | `MONITORING_HOST` | `PEER_PORTS` |

Allow rules are inserted at the top of the chain, the per-port catch-all `DROP` is
appended, so a packet is tested against every exception before being dropped.

`DOCKER_PORTS` is LAN-wide because those services are meant to be browsed. That is a
network boundary, not an authentication one — each workload carries its own login.

`PEER_PORTS` is the tighter tier, for ports with no meaningful authentication of their
own. **Never move an entry from `PEER_PORTS` to `DOCKER_PORTS`.** `node-exporter` is a
full inventory of the host and `alloy` mounts `docker.sock`.

Every rule is scoped `-i eth0`, so it applies to packets arriving from outside the box.
Container-to-container traffic on a Docker bridge also traverses `DOCKER-USER` — Docker
enables bridge-netfilter — and a rule without `-i eth0` would drop that too. This bit
`network-stack` live during its own firewall rollout.

## Egress containment

The lab may reach the internet. It may not reach the LAN or the VPN subnet, apart from
three exceptions.

| Flow | Rule |
| --- | --- |
| Established/related replies | `RETURN` — first, and load-bearing |
| DNS to `NETWORK_HOST` | `RETURN` on `udp/53`, `tcp/53` |
| Alloy → Loki on `MONITORING_HOST` | `RETURN` on `tcp/3100` |
| Anything else to `192.168.1.0/24` | `DROP` |
| Anything else to `10.8.0.0/24` | `DROP` |
| Anything else (the internet) | Not matched, so allowed |

Ordering is the reverse of the ingress rules: exceptions are inserted at the top, the
two catch-all drops appended at the bottom.

**The conntrack rule is the one whose absence is confusing.** Without it, the catch-all
drops kill the reply path of every connection a LAN browser opens to a workload, and the
symptom is "the workload is down", not "egress is blocked". It is inserted first for
that reason.

**The rules match `-s 172.22.0.0/16`**, the `lab-stack` bridge subnet, pinned in
`compose.yml`'s `ipam` block. That is what makes them apply to lab workloads and nothing
else — and it means the subnet is one fact stored in two files. Change it in one place
and containment stops applying with no error at all. The egress test in
`docs/operations.md` is the only thing that catches it.

## Adding an exception

When a workload genuinely needs to reach a LAN host — a printer, a NAS — add one line,
scoped to that host and port, with a comment saying why:

```bash
# <workload> needs <thing> on the NAS for <reason>
lab_egress_allow tcp 445 192.168.1.50
```

Never widen an exception to a subnet, and never remove the catch-all drops. An
experiment that needs broad LAN access is one that should be reviewed rather than
sandboxed.

## Verifying

The ingress side can be checked with `iptables -L DOCKER-USER -n -v --line-numbers`. The
egress side cannot be read off the chain with confidence — the rules are correct-looking
whether or not the subnet still matches — so test it from inside a container. The
commands are in [../docs/operations.md](../docs/operations.md#testing-egress-containment).

## IPv6

`ip6tables` blanket-drops every published port. `eth0` has no globally-routable IPv6
today, so none of it is reachable; the rules exist so that if the router ever starts
advertising IPv6 and this host picks up a global address, these ports fail closed
instead of silently becoming internet-reachable. There is no LAN/VPN IPv6 prefix to
allowlist, hence a blanket drop rather than allow-then-drop.

Note the egress rules are IPv4-only. If IPv6 ever arrives on this LAN, they need an
`ip6tables` counterpart — the blanket ingress drops do not cover a workload reaching
*out* over IPv6.
