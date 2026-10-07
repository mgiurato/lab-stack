# Workloads

One directory per experiment, holding only gitignored state. The service definition
lives in `compose.yml`; this directory is where its volumes land.

Everything here is disposable. Nothing outside this stack may depend on it, and its data
is not backed up.

## Adding one

1. **Pick the next free port from 8100.** 8090–8092 are the docs servers; the other
   stacks own everything below that. Keep the container's own port on the right-hand
   side of the mapping, e.g. `"8101:3000/tcp"`.

2. **Add the service to `compose.yml`** under the Workloads heading, with:
   - a **pinned** image tag — never `latest`, never a bare major;
   - `security_opt: [no-new-privileges:true]`;
   - `networks: [lab-stack]`;
   - `logging: *default-logging`;
   - volumes under `./workloads/<name>/`, which `.gitignore` already covers;
   - **its login enabled**, with the credentials sourced from `.env` as required
     variables. If the workload has no authentication at all, say so in a comment —
     that is a fact worth being explicit about rather than a gap to leave implied.

3. **Add the port to `DOCKER_PORTS`** in `firewall/apply-rules.sh` and re-run it. A
   published port with no allow rule is unreachable.

4. **Optionally add a `*.lan` hostname** — the DNS record and Caddy site block live in
   `network-stack`; the recipe is in
   [../docs/networking.md](../docs/networking.md#adding-a-lan-hostname). Put
   `basic_auth` on the site block regardless of the workload's own login.

5. **Add a card** to documentation-stack's `caddy/site/index.html` if it is something
   you will actually reach for.

Then test it, including the egress check in
[../docs/operations.md](../docs/operations.md#testing-egress-containment) if the
workload is expected to talk to anything.

## Things that will bite

**It cannot reach the LAN.** By design — see
[../firewall/README.md](../firewall/README.md#egress-containment). A workload that needs
a printer, a NAS or another host needs a one-line, host-and-port-scoped exception, not
the removal of the drops.

**Docker creates a missing bind-mount source as a directory.** If a container fails to
start with a mount error, that is the first thing to check.

**The reverse proxy is not the workload's authentication.** The published port on
`LAB_HOST` is reachable directly from any LAN device, whatever Caddy does in front.

**A workload that stops being disposable should leave.** The signals and the move are in
[../docs/architecture.md](../docs/architecture.md#promotion).

## Current workloads

| Name | Port | What it is |
| --- | --- | --- |
| `searxng` | 8100 | Metasearch engine, queried by LAN browsers. It has no login, and holds no accounts or stored queries; it runs without the limiter, so without Valkey. Config in `workloads/searxng/config/`, cache in a named volume |
| `snapotter` | 8101 | File processing for images, video, audio, PDF and documents, with local AI (OCR, background removal, upscaling). Three containers — the app, PostgreSQL (`snapotter-db`) and Redis (`snapotter-redis`) — with its login on |

**SnapOtter's limits are lowered from upstream's.** Upstream caps the app at 6 GB,
PostgreSQL at 1 GB and Redis at 512 MB (7.5 GiB); this stack runs in a 4 GiB LXC, so
they are 2 GB, 512 MB and 256 MB. The app refuses its heaviest modes at 2 GB (HQ erase
wants 8 GB) and may fail a large video. Restore upstream's limits when the LXC has 8 GiB.
Its AI models are kept in `snapotter-data`, so disk grows with use.

**SnapOtter 2.2.0 connects to PostgreSQL as the database owner.** The least-privilege
runtime role arrived in 2.3.0; upstream's `main` compose file already uses it, and with
2.2.0 the app exits at boot because the role it logs in as does not exist.

**SnapOtter's telemetry is switched off** with `SNAPOTTER_TELEMETRY: "0"` in
[compose.yml](../compose.yml). The official image otherwise reports usage analytics (PostHog)
and initialises Sentry for error reports, and this stack's egress to the internet is open.
The variable overrides the in-app toggle, so nothing needs doing at sign-in; keep it when
upgrading.

Its login is seeded from `SNAPOTTER_PASSWORD` and the app forces a new one at the first
sign-in. State is in named volumes, so `docker compose down -v` wipes it.

Both images are pinned: SearXNG to its dated tag, SnapOtter to `2.2.0`, PostgreSQL and
Redis to the digests upstream pins.

Other ideas this stack was built for, none of them deployed: a local LLM (size the LXC for
it first — model weights dominate both RAM and disk), a paperless document archive,
whatever the last interesting link was. Something whose data is wanted, such as a recipe
manager or a food diary, does not belong here: the migration plan in `network-stack`
puts those in `health-stack`.
