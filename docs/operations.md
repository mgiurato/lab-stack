# Operations — lab-stack

## Joining the lab to the rest of the lab

Four edits outside this repo, once, when the stack is first deployed. None of them grant
the lab any privilege — they let the lab be reached, scraped and read.

**1. `network-stack` — reverse proxy.** Add `LAB_HOST=<address>` to `.env` and to the
`caddy` service's `environment:` block in `compose.yml`, then a site block per workload
in `proxy/caddy/Caddyfile`:

```
http://pdf.lan {
    basic_auth { admin {$CADDY_BASIC_AUTH_HASH} }
    reverse_proxy {$LAB_HOST}:8100
}
```

**2. `network-stack` — DNS.** A record in `dns/pihole/etc-dnsmasq.d/lan.conf` pointing
the hostname at network-stack's own address, not at `LAB_HOST`.

**3. `monitoring-stack` — scraping.** Two jobs in `prometheus/prometheus.yml.tmpl`,
using a `__LAB_HOST__` placeholder substituted by `docker-entrypoint.sh` exactly as the
other hosts are:

```yaml
  - job_name: node-exporter-lab
    basic_auth:
      username: prometheus
      password_file: /etc/prometheus/secrets/lab-stack-scrape-password
    static_configs:
      - targets: ["__LAB_HOST__:9100"]

  - job_name: alloy-lab
    static_configs:
      - targets: ["__LAB_HOST__:12345"]
```

Add `LAB_HOST` to that repo's `.env`, and the lab's address to **`LOG_SHIPPER_HOSTS`**
in its `firewall/apply-rules.sh` so Alloy may push to Loki. **Not to `PEER_HOSTS`** —
that list grants the right to read every metric and silence every alert, and the lab has
no business with either.

**4. `documentation-stack` — docs.** An alias entry in `caddy/docs-site/index.html`
(`'/lab-stack/(.*)': 'http://<LAB_HOST>:8092/$1'`, plus the bare `/lab-stack/` → README
entry) and a section in `caddy/docs-site/_sidebar.md`.

## Verifying a deployment

```bash
# This stack
docker compose ps
curl -s localhost:8092/README.md | head -1
docker exec alloy sh -c 'wget -qO- localhost:12345/-/ready'
curl -s -o /dev/null -w '%{http_code}\n' localhost:9100/metrics    # 401 — auth required

# From monitoring-stack
curl -s localhost:9090/api/v1/targets \
  | python3 -c "import json,sys; [print(t['health'], t['labels']['job']) for t in json.load(sys.stdin)['data']['activeTargets'] if 'lab' in t['labels']['job']]"
curl -s 'localhost:3100/loki/api/v1/query?query={stack=\"lab-stack\"}' | head -c 200
```

### Testing egress containment

The part that is easy to assume and be wrong about. Run it from **inside a workload
container**, because that is the only place the rules apply:

```bash
# Should FAIL (timeout) -- the LAN is off limits
docker exec stirling-pdf sh -c 'timeout 5 wget -qO- http://192.168.1.39/admin ; echo "exit=$?"'
docker exec stirling-pdf sh -c 'timeout 5 wget -qO- https://192.168.1.20:8006 ; echo "exit=$?"'

# Should SUCCEED -- the internet is open, and these two exceptions exist
docker exec stirling-pdf sh -c 'timeout 8 wget -qO- https://example.com >/dev/null; echo "exit=$?"'
docker exec alloy sh -c 'timeout 5 wget -qO- http://$MONITORING_HOST:3100/ready'
```

A workload image without `wget` or `timeout` can be tested with a throwaway instead:

```bash
docker run --rm --network lab-stack alpine:3.20 \
  sh -c 'timeout 5 wget -qO- http://192.168.1.39/admin; echo "exit=$?"'
```

An `exit=0` on either of the first two means the containment is not in effect. The usual
cause is the bridge subnet in `compose.yml` no longer matching `LAB_SUBNET` in
`firewall/apply-rules.sh` — they are one fact written in two files, and a mismatch fails
silently rather than erroring.

## Adding a workload

See [workloads/README.md](../workloads/README.md).

## Removing a workload

Deliberately trivial, because it should happen often:

```bash
docker compose stop <name> && docker compose rm -f <name>
rm -rf workloads/<name>/
```

Then delete its service from `compose.yml`, its port from `DOCKER_PORTS` in
`firewall/apply-rules.sh`, and — if it had one — its site block and DNS record in
network-stack. Nothing else should reference it; if something does, that is the bug.

## Troubleshooting

**A workload can't reach something on the LAN.** Working as designed. If the flow is
genuinely needed, add a commented `lab_egress_allow` line for that exact host and port
in `firewall/apply-rules.sh`. Don't remove the catch-all drops.

**A workload is unreachable from the LAN.** Check `DOCKER_PORTS` contains its port and
re-run `./firewall/apply-rules.sh`. A published port with no allow rule is dropped by the
catch-all for that port.

**Prometheus reports the lab's targets down.** Check the credential first: the hash in
`monitoring/web-config/node-exporter.yml` and the plaintext in monitoring-stack's
`prometheus/secrets/lab-stack-scrape-password` must match, and that plaintext file must
be owned `65534:65534` mode `400` — Prometheus reads it as `nobody` on every scrape, and
a root-owned `600` file produces `unable to read basic auth password`, which reads like
a wrong password rather than a wrong mode.
