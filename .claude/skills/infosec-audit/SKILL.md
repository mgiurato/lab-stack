---
name: infosec-audit
description: Perform an independent infosec audit of lab-stack, the low-trust sandbox LXC — verifies that egress containment actually holds, that no workload has become load-bearing, and the usual exposure/secrets/hardening checks, against the live system rather than the docs.
---

# Infosec Audit

Audit this repository and the live host it manages as an external security reviewer
would: skeptical of the docs, verifying claims against actual state, and looking for
gaps the maintainer wouldn't think to document because they're used to them.

`CLAUDE.md` and `docs/security.md` already describe the *intended* trust model and a
list of *known* gaps. Do not just restate those. Your job is to (a) verify they're still
true against the live system, and (b) find what they missed.

## Scope

This repo is the `lab-stack` Docker Compose project: a low-trust sandbox in its own
unprivileged Proxmox LXC, running disposable experiments plus this host's two agents
(`node-exporter`, `alloy`) and a read-only docs server.

**Audit this project only.** `network-stack`, `monitoring-stack` and
`documentation-stack` are separate Compose projects in their own repos, each with its own
audit log. Findings about those belong there — but do note anything about *this*
project's interface to them: the peer-only agent ports (9100, 12345), whether the lab has
been granted an entry in another stack's `PEER_HOSTS` (it must not have been), and
whether anything outside this repo has come to depend on a workload here.

**This stack inverts the usual question.** Elsewhere you ask whether a service is
adequately protected from the LAN. Here, also ask the reverse: whether the LAN is
adequately protected from this stack. Two checks have no counterpart in the other repos
and matter more than anything else in this one:

1. **Egress containment actually holds.** Do not read it off the `DOCKER-USER` chain —
   the rules look correct whether or not they still match. Test it from inside a
   container, per `docs/operations.md`. The specific silent failure is the bridge subnet
   in `compose.yml`'s `ipam` block drifting from `LAB_SUBNET` in
   `firewall/apply-rules.sh`; they are one fact in two files. Check they agree, then
   prove it empirically anyway.
2. **No workload has become load-bearing.** Check whether any workload has acquired a
   dependent outside this stack, data that would be missed, an alert, or a LAN egress
   exception that is no longer obviously temporary. Each is a finding: the sandbox's
   safety properties rest on everything in it being disposable, and that erodes quietly
   rather than all at once.

Also verify every workload with an authentication option has it enabled, and that each
is reachable directly on its published port — Caddy's `basic_auth` gates only the
browser route, so it is not the workload's authentication.

Router-level exposure is documented, not directly inspectable from here — flag anything
that depends on router configuration as "assumed, not verified."

## Method

Work through these in order. Use Bash freely (read-only commands only — this is an
audit, not a remediation pass unless the user asks you to fix what you find).

### 1. Baseline from docs

Read `docs/security.md`, `docs/networking.md`, `docs/proxmox.md`, and each component
`README.md` (`firewall/`, `monitoring/`, `workloads/`). Note every claim that's checkable: which ports are said to be exposed,
which services are said to require auth, which secrets are said to be excluded from Git,
image pinning claims, capability grants.

### 2. Verify exposure claims

- `docker compose ps` / `docker port <container>` for every published port — cross-check
  against what `compose.yml` and the docs say should be published.
- Re-derive the compose port list yourself from `compose.yml` rather than trusting the
  table in `docs/security.md` — tables drift from the file that actually governs
  behavior.
- Check `iptables -L -n` / `iptables -t nat -L -n` inside the container for anything
  beyond Docker's own chains.
- Router-level forwarding cannot be verified from inside the container — call this out
  explicitly as an assumption rather than a confirmed fact.

### 3. Secrets handling

- Confirm `.env`, `monitoring/web-config/node-exporter.yml` and the workload state
  directories listed in `.gitignore` are genuinely untracked: `git ls-files` filtered
  against those paths, or `git check-ignore -v <path>`. Workload directories are the
  likely offender here — they are added ad hoc and the ignore rule is a glob, so a
  workload that puts state somewhere unexpected escapes it.
- Search full git history for material that shouldn't be there, not just the working
  tree: `git log --all -p -- '*.env' '*.pem' '*.key' '*.conf'` and a broad
  `git log --all -S PASSWORD -p` / `-S BEGIN` sweep for anything that looks like a
  credential or private key ever committed, even if later removed. Removed-but-present
  history is exactly how the previously-rotated Pi-hole TLS key and `cli_pw` leaks
  happened in `network-stack` — check whether this repo has acquired any of its own.
  Workload credentials seeded from `.env` are the plausible source.
- Check current file permissions on anything sensitive that exists on disk (`.env`,
  `monitoring/web-config/node-exporter.yml`, any workload's config directory) —
  world-readable secrets are a real risk even if Git is clean.
- Check whether any `.env` value duplicates a value already committed elsewhere (e.g. a
  password reused across `PIHOLE_PASSWORD` / `GRAFANA_ADMIN_PASSWORD`) — read the actual
  running values only if necessary for a yes/no comparison, never print them into your
  output.

### 4. Container hardening

For every service in `compose.yml`: check `cap_add`, `privileged`, host mount scope
(`/`, `/var/run/docker.sock`, `/proc`, `/sys`), and network mode. Ask for each one
whether the grant is as narrow as the service's actual function requires — e.g.
`alloy` mounting the Docker socket and `node-exporter` mounting `/` are functional
requirements for those tools, but confirm nothing broader crept in. Apply the most
scepticism to the **workloads**: they are added casually, and a `privileged: true` or a
host mount copied from an upstream README is exactly the kind of thing that arrives
without being noticed. No workload has any business with `docker.sock`, a host path
outside its own `workloads/<name>/` directory, or `network_mode: host` — the last of
which would bypass egress containment entirely, since the rules match on the bridge
subnet.

### 5. Trust boundary and lateral movement

- Enumerate which services have zero authentication of their own (per the docs and by
  actually requesting their endpoints, e.g. `curl -s -o /dev/null -w '%{http_code}'` against
  each `*.lan` vhost) and cross-check that list is still accurate — services get added to
  the Caddyfile over time and may not all be reflected in the docs.
- The peer-only ports (9100, 9617, 12345, 9180) have no authentication at all. Verify
  they are in `PEER_PORTS`, not `DOCKER_PORTS`, and that Caddy's admin API on 2019 is
  still loopback-only and unpublished.
- Check whether any proxied backend (the `*arr` apps, etc.) is reachable *directly* by IP
  and port, bypassing Caddy and any future auth added there — reverse-proxy auth is
  meaningless if the origin is still reachable on its own port from the same LAN segment.

### 6. Transport security

Confirm what `docs/security.md` documents about Caddy's internal CA and plain-HTTP
backend hops is still what's actually configured in `proxy/caddy/Caddyfile` — internal
CA usage and `auto_https` settings are easy to silently regress when a new site block is
added by copy-paste from an old one.

### 7. Update hygiene and supply chain

- List every image tag in `compose.yml` and flag moving tags (`latest`, bare majors like
  `caddy:2`) versus pinned ones. Workloads are where an unpinned tag will show up, and
  an unvetted image on a moving tag is the worst combination in this repo.
- Check any workload Dockerfile or entrypoint for unpinned upstream downloads
  (tarballs, models or scripts fetched by URL with no checksum verification).
- If the `docker.sock`-mounted container (`alloy`) is running a moving tag, flag that
  specifically — a supply-chain compromise there has more reach than in a workload.

### 8. Backup and recovery

Cross-check `docs/proxmox.md`'s backup and restore claims against what's actually
verifiable from here: does a restore drill appear to have been exercised recently (per
the doc's own "last verified" notes), and does the backed-up data actually cover every
untracked-but-load-bearing path (`.env`, `monitoring/web-config/node-exporter.yml`)?
Workload state is deliberately *not* backed up — if a backup job has started covering it,
that is a finding in itself, because it means something in there is being treated as
worth keeping. A backup that misses WireGuard's `wg0.json` (client definitions) is a common
silent gap — check whether that's specifically covered.

### 9. Monitoring and detection

Availability monitoring existing is not the same as security monitoring existing. Check
whether there's any detection for: repeated auth failures against a workload's login,
a workload's outbound traffic volume, or a container in this stack being restarted or
replaced outside a deploy. If none exists, that's a
finding (lack of detection ≠ lack of risk), not something to silently accept because it's
out of the stack's original scope.

## Reporting

Grade every finding: **Critical / High / Medium / Low / Informational**, using standard
judgment (network exposure + auth bypass potential + ease of exploitation from an
assumed-hostile LAN device, since that's this stack's actual threat model per
`docs/security.md`'s Trust Model section — not an internet-facing web-app threat model).

For each finding give: what you checked, what you found (with the actual command output
or file evidence, not a paraphrase), why it matters given this stack's specific trust
model, and a concrete remediation — pointing at the specific file/line to change when
applicable (`compose.yml`, `proxy/caddy/Caddyfile`, `.gitignore`, etc.).

Explicitly list what you verified as **already fine** — don't just produce a list of
problems; confirm the things the docs claim are true and checked out (e.g. "confirmed:
neither `.env` nor any workload's state appears anywhere in git history"). A silent audit that
only mentions problems is indistinguishable from an incomplete one.

Do not modify any files, rotate any credentials, or change any running configuration
during the audit itself. If you find something urgent (e.g. a live unclaimed admin
panel, a currently-forwarded port that shouldn't be), say so clearly and ask before
acting — remediation is a separate, explicit step from the audit.
