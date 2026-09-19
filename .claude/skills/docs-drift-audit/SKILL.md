---
name: docs-drift-audit
description: Audit this repo's docs (CLAUDE.md, docs/*.md, component READMEs) against the live system and config files, and report every place a doc's claim no longer matches reality.
---

# Docs Drift Audit

This repo runs on the principle that `CLAUDE.md`, `docs/*.md`, and each component
`README.md` are the source of truth an operator (or a future Claude session) reads
*before* the config files. That only works if the docs are still true. Config changes
fast — compose edits, Caddyfile site blocks, `.env` additions, upstream image updates —
and docs don't update themselves. This audit finds where they've diverged.

This is not the `infosec-audit` skill — that one grades security risk. This one grades
**accuracy**: does the doc still describe what's actually running, regardless of whether
the drift itself is a security problem. A stale doc about a perfectly safe config is
still a finding here.

## Scope

Every doc file: `CLAUDE.md`, `docs/architecture.md`, `docs/networking.md`,
`docs/operations.md`, `docs/proxmox.md`, `docs/security.md`, and each component
`README.md` (`dns/`, `proxy/`, `vpn/`, `ddns/`, `certificates/`, `firewall/`,
`monitoring/`). Cross-check each against: `compose.yml`, `.env` / `.env.example`,
`proxy/caddy/Caddyfile`, `dns/pihole/etc-dnsmasq.d/lan.conf`, `monitoring/alloy/config.alloy`,
`firewall/apply-rules.sh`, `docs-server/Caddyfile`, other config files the docs reference by
name, and live system state where checkable (`docker compose ps`, `docker port`, `curl`
against `*.lan` hosts, `iptables -L DOCKER-USER -n`).

**This repo only.** CT 103 runs three Compose projects in three repositories; each has its
own copy of this skill and its own docs. A claim here about another stack is in scope only
where this repo is the one making it — e.g. a stale peer address, or a cross-repo link
that no longer resolves.


## Method

### 1. Extract checkable claims

Read every doc file above. Pull out every claim that is a *fact about the system*, not
prose/rationale — port numbers, variable counts and names, file paths, "requires
auth"/"no auth" statements, image tags, container names, directory structures, dates
("last verified"), counts ("all N variables", "N services"). Ignore claims that are
inherently unverifiable from here (e.g. router-level config) — flag those separately as
"unverifiable, not drift."

### 2. Cross-check against config as the source of truth

For each extracted claim, find the file that actually governs that behavior and compare:

- Port claims → `compose.yml`'s `ports:` blocks, live `docker port <container>`.
- Hostname/routing claims → `proxy/caddy/Caddyfile` site blocks, `lan.conf` DNS records.
- "No auth" / "has auth" claims → actually request the endpoint
  (`curl -s -o /dev/null -w '%{http_code}\n'`) rather than trusting the doc or the last
  audit that made the claim; auth can be added or removed by an unrelated change (e.g. an
  upstream image update giving a service its own login) without anyone updating the
  trust-model doc.
- Variable/count claims (e.g. "`.env` has N variables") → count them yourself in the
  actual file right now.
- Image tag claims → `compose.yml` again, not the doc's memory of the last time someone
  looked.
- Directory/file path claims → confirm the path exists (`ls`) — renamed or removed paths
  are a common silent-drift source when a service gets refactored.
- "Last verified"/dated claims → these are supposed to go stale; the finding isn't that
  they're old, it's whether anything *material* has changed since that date that the note
  doesn't account for (compare against `git log` for that file/config since the noted
  date).

### 3. Cross-doc consistency

Some facts are documented in more than one place (e.g. the exposure/port table appears in
both `docs/security.md` and `docs/networking.md`). Check these agree with each other,
not just with the config — two docs silently disagreeing is drift even if neither one is
individually wrong yet.

### 4. Open-issue content leaking into pure-documentation files

`docs/*.md` and every component `README.md` are pure documentation of the system as it
stands — no "Known Gaps," "Closed," or similar running open/closed lists belong in them.
Open findings live only in `docs/audits/infosec-audit-repo.md`. If a doc file has grown
a gaps/closed-style section, that's drift too: flag it for the finding to be moved to the
audit log and the doc trimmed back to description. Separately, check whether anything
already in the audit log has actually been closed by a change that didn't update it, and
vice versa — a finding marked closed that has since regressed (e.g. a hash moved back
into a tracked file, a port re-exposed).

### 5. CLAUDE.md-specific check

`CLAUDE.md` is special: it's loaded into every session, so drift there compounds across
all future work, not just this one. Verify its command list still runs as described
(spot-check a couple, don't need to run all), its architecture diagram still matches
`compose.yml`'s actual service graph, and its "gotchas" section doesn't contradict
current behavior (e.g. a workaround for a bug that's since been fixed upstream).

## Reporting

For each drift finding: quote the stale doc claim (file + line), show the current actual
state (command output or file evidence), and give the specific edit to make the doc
correct again — point at the exact line to change, don't just describe the problem.

Group findings by severity of consequence, not security risk:

- **Misleading** — doc claim is actively wrong in a way that would cause a bad decision
  (e.g. tells you a service has no auth when it now does, so you'd add redundant
  `basic_auth`; or the reverse, where you'd skip adding auth because the doc says it's
  already covered).
- **Stale** — claim is out of date but low-consequence (a "last verified" date, a variable
  count) — still worth fixing, lower urgency.
- **Cosmetic** — drift with zero decision impact (a stale example value, a typo'd
  container name in prose).

Explicitly list what you checked and found **still accurate** — this audit is only
useful if it's exhaustive, not just a list of problems. A pass that only reports drift is
indistinguishable from one that didn't check most of the docs.

Do not silently fix anything while auditing — report first. Fixing docs is cheap and
low-risk once findings are confirmed, so it's reasonable to fix immediately after
reporting if the user says to (unlike `infosec-audit`, which treats remediation of live
config as a separate, explicit step) — but the audit pass itself stays read-only so the
report reflects the true starting state.
