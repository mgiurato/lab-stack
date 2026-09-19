# Monitoring — lab-stack

This stack runs two agents and no servers. Prometheus, Alertmanager, Grafana, Loki and
Uptime Kuma all live in `monitoring-stack`; the agents here measure *this host* and
cannot be centralised.

| Agent | Port | Measures | Auth |
| --- | --- | --- | --- |
| `node-exporter` | 9100 | CPU, memory, disk, network of this LXC | Basic auth |
| `alloy` | 12345 | Container logs + the host journal, pushed to Loki | **None** |

Both ports are in `PEER_PORTS` — reachable from `MONITORING_HOST` only. Don't add a
server here, and don't move an agent to monitoring-stack.

## Scrape credential

`node-exporter` requires basic auth because its port is LAN-published and its output is
a complete inventory of the host. One identity per host:

- The bcrypt hash goes in `monitoring/web-config/node-exporter.yml` (gitignored; copy
  `node-exporter.yml.example`). Do **not** escape `$` there — that file is not
  interpolated.
- The matching plaintext goes in monitoring-stack's
  `prometheus/secrets/lab-stack-scrape-password`, owned `65534:65534` mode `400`.

Generate one with:

```bash
docker run --rm caddy:2 caddy hash-password --plaintext 'your-password'
```

`alloy` has no authentication option at all, so for it the firewall allowlist is the
only control. That matters more here than elsewhere: Alloy mounts `docker.sock`
read-only, which makes it the highest-value target in this stack. See
[../docs/security.md](../docs/security.md).

## Logs

Every log stream is labelled `stack="lab-stack"`, both container logs and the host
journal. Without that label a container called `alloy` here and one on another host
collapse into a single Loki stream, and an alert firing on repeated SSH failures cannot
say which machine it means.

The host journal is worth shipping here specifically: this is the machine most likely to
be running something unvetted, so the record of who logged into it is the record that
matters.

```bash
# from monitoring-stack
curl -s 'localhost:3100/loki/api/v1/query?query={stack="lab-stack"}' | head -c 400
```

## What is not monitored

Workloads themselves have no scrape jobs and should not get alerts. They are disposable;
an experiment being down is not an incident. `node-exporter` covers the host they run
on, which is the part that matters — if a workload exhausts the LXC's memory, that shows
up there.

A workload that warrants its own alert has outgrown this stack — see
[../docs/architecture.md](../docs/architecture.md#promotion).
