# Operations — lab-stack

## Joining the lab to the rest of the lab

Four edits outside this repo, once, when the stack is first deployed. None of them grant
the lab any privilege — they let the lab be reached, scraped and read.

**1. `network-stack` — reverse proxy.** Add `LAB_HOST=<address>` to `.env` and to the
`caddy` service's `environment:` block in `compose.yml`, then a site block per workload
in `proxy/caddy/Caddyfile`:

```
http://search.lan {
    basic_auth { admin {$CADDY_BASIC_AUTH_HASH} }
    reverse_proxy {$LAB_HOST}:8100
}

http://files.lan {
    reverse_proxy {$LAB_HOST}:8101
}
```

Each has an `https://` twin that adds `tls internal`, and a matching `@search` /
`@files` handler in the `*.{$DUCKDNS_LAN_DOMAIN}.duckdns.org` site block, so
`search.pelae-lan.duckdns.org` and `files.pelae-lan.duckdns.org` work with any client
resolver. Only SearXNG's routes carry `basic_auth`: it has no login of its own, while
SnapOtter has one and sends its own `Authorization` header, which a proxy-level login
would collide with.

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

**4. `documentation-stack` — docs.** That stack's Caddy reverse-proxies this docs-server
server-side, so the browser only ever talks to `docs.lan`:

- an alias pair in `caddy/docs-site/index.html` —
  `'/lab-stack/(.*)': '/peer-docs/lab-stack/$1'` and
  `'/lab-stack/': '/peer-docs/lab-stack/README.md'`;
- a `handle_path /peer-docs/lab-stack/*` block in its `caddy/Caddyfile`,
  reverse-proxying to `{$LAB_HOST}:8092`;
- `LAB_HOST` in its `.env` and `.env.example`, and passed through `compose.yml`'s
  `caddy.environment`;
- a section in `caddy/docs-site/_sidebar.md`.

## Verifying a deployment

```bash
# This stack -- on CT 104, in /root/lab-stack
docker compose ps
curl -s localhost:8092/README.md | head -1
curl -s localhost:12345/-/ready                                    # "Alloy is ready." (its image has no wget/curl)
curl -s -o /dev/null -w '%{http_code}\n' localhost:9100/metrics    # 401 — auth required

# Logs reaching Loki -- a non-zero count. loki-proxy requires a login; the plaintext is
# this stack's LOKI_PUSH_PASSWORD, read from .env so it is never echoed
curl -sG -u "loki:$(grep '^LOKI_PUSH_PASSWORD=' .env | cut -d= -f2-)" \
  "http://$(grep '^MONITORING_HOST=' .env | cut -d= -f2-):3100/loki/api/v1/query" \
  --data-urlencode 'query=sum(count_over_time({stack="lab-stack"}[5m]))'

# From monitoring-stack -- on CT 106, in /root/monitoring-stack. Prometheus requires a
# login; the credential is read from its secrets file, never typed or printed
curl -s -u "monitoring:$(cat prometheus/secrets/internal-auth-password)" localhost:9090/api/v1/targets \
  | python3 -c "import json,sys; [print(t['health'], t['labels']['job']) for t in json.load(sys.stdin)['data']['activeTargets'] if 'lab' in t['labels']['job']]"
```

The last one should print `up node-exporter-lab` and `up alloy-lab`.

### Testing egress containment

The part that is easy to assume and be wrong about. Run it from **a container on the
`lab-stack` network**, because that is the only place the rules apply. The workload
images are no use for it — `alloy` has neither `wget` nor `curl`, and `searxng` has no
`timeout` — so use a throwaway Alpine container, which gets an address in
`172.22.0.0/16` like any workload:

```bash
# Should FAIL with exit=143 (killed by the timeout) -- the LAN is off limits
docker run --rm --network lab-stack alpine:3.20 sh -c 'timeout 5 wget -qO- http://192.168.1.39/admin >/dev/null; echo "exit=$?"'
docker run --rm --network lab-stack alpine:3.20 sh -c 'timeout 5 wget -qO- --no-check-certificate https://192.168.1.20:8006 >/dev/null; echo "exit=$?"'

# Should SUCCEED -- the internet is open, and DNS and Loki are the two LAN exceptions
docker run --rm --network lab-stack alpine:3.20 sh -c 'timeout 8 wget -qO- https://example.com >/dev/null; echo "exit=$?"'
docker run --rm --network lab-stack alpine:3.20 sh -c 'timeout 5 nslookup example.com 192.168.1.39 >/dev/null; echo "exit=$?"'
docker run --rm --network lab-stack alpine:3.20 sh -c 'timeout 5 wget -qO- http://192.168.1.70:3100/ready 2>&1; echo "exit=$?"'

docker image rm alpine:3.20    # leave nothing behind
```

The Loki line passes with `server returned error: HTTP/1.1 401 Unauthorized` and
`exit=1`: loki-proxy asks for a login even on `/ready`, and getting that answer at all
means the packet got through. Only `exit=143` there means it was dropped.

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
