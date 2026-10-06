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
| `stirling-pdf` | 8100 | Self-hosted PDF editing — split, merge, OCR, convert. The worked example; delete it if you don't want it |

Planned, replacing the Stirling-PDF example before the first deploy; the compose file does
not have them yet:

- **SearXNG**, a metasearch engine (`searxng/searxng`; the container listens on 8080,
  configuration lives in `/etc/searxng`, its cache in `/var/cache/searxng`, and settings
  can be given as `SEARXNG_*` variables).
- **SnapOtter**, a file-processing toolbox for images, video, audio, PDF and documents,
  with local AI for OCR, background removal and upscaling (`snapotter/snapotter`, port
  1349). It runs with PostgreSQL and Redis beside it. Its own compose file caps the app
  at 6 GB of RAM, PostgreSQL at 1 GB and Redis at 512 MB, so with those limits this stack
  needs 8 GiB, not the 4 GiB of [proxmox.md](../docs/proxmox.md); the AI models are kept
  in its data volume, so disk grows too. It ships with the login `admin`/`admin`, which
  falls under the authentication rule above.

Both are pinned to a release tag when they are added, as every image here is.

Other ideas this stack was built for, none of them deployed: a local LLM (size the LXC for
it first — model weights dominate both RAM and disk), a paperless document archive,
whatever the last interesting link was. Something whose data is wanted, such as a recipe
manager or a food diary, does not belong here: the migration plan in `network-stack`
puts those in `health-stack`.
