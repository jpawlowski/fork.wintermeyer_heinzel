---
name: heinzel-fleet-audit
argument-hint: "[hostname1 hostname2 ...]"
description: Compare key policies across all servers in
  memory/servers/ to surface silent drift. Makes no configuration
  changes; writes one audit-trail line to each host's journal.
  Probes unattended-upgrades, sshd effective config, firewall
  posture, MTA, time sync, and auto-reboot behaviour. Use when
  the user asks to "fleet audit", "vergleiche alle server",
  "policy drift check", "are my servers configured the same?",
  or after a fix on one host to find which others carry the
  same bug.
---

# heinzel-fleet-audit

Cross-server policy audit. Heinzel knows every host
individually but has nothing that holds hosts against each
other. This skill closes that gap by probing the same set of
settings on every server in `memory/servers/` and rendering a
side-by-side comparison so silent drift becomes visible.

**No configuration changes.** The audit never alters any
host's configuration. The only write is a single audit-trail
line to each host's system journal (step 6 below). Acting on
findings is a separate step (heinzel-housekeeping for
per-host fixes, or manual edits with explicit user approval).

**Never run automatically** — only on explicit user request.

## When to use

- "Run a fleet audit"
- "Vergleiche die Policies auf allen Servern"
- "Drift check across the fleet"
- After fixing a config bug on one host: "which other hosts
  have the same problem?"

Do NOT auto-invoke for generic phrases like "check my
servers" — that maps to single-host housekeeping.

## Workflow

1. **Discover hosts.** By default every server in
   `memory/servers/` (`--all` below). The user may pass an
   explicit subset as arguments — in that case audit only
   those.

2. **Onboard all hosts in one call.** The pipeline in
   `rules/first-connection.md` applies to every host; run its
   remote part for all of them at once, as
   `rules/multi-host.md` → Onboarding for a fan-out says:

   ```bash
   bin/heinzel-fanout --onboard host1 host2 host3
   ```

   Use `--all` instead of host names when the user did not
   name a subset. Carry each skip line into the report's
   `Skipped:` header; do not prompt for missing users.

3. **Probe in parallel.** For each in-scope host, run the
   probes from `references/probes.md` in a single batched
   SSH command, with the standard options from `CLAUDE.md` →
   SSH Options.
   Hosts that time out or refuse the connection go on a
   "skipped: unreachable" list.

   Do it for all hosts in one `--read` fan-out, which applies
   those options, and attach the journal line. The script is
   the body of the probe block in `references/probes.md`,
   without its `ssh … '` wrapper:

   ```bash
   bin/heinzel-fanout --read --max-lines 0 \
     --log '[<operator> as {user}] read-only: fleet-audit probe' \
     host1 host2 host3 <<'EOS'
   <probe script from references/probes.md>
   EOS
   ```

   Then read each host's raw output from the run directory
   (`out/<host>`) the helper prints, and split it on the
   `###<key>###` markers. Hosts listed as `unreach` are the
   unreachable ones.

4. **Render comparison.** Build one table per probe category
   using the format in `references/output-format.md`. Hosts
   are columns, settings are rows. Cells that differ across
   columns get visual emphasis.

5. **Surface drift.** After the tables, emit a short "Drift
   detected" section that lists each disagreement and the
   recommended fix (link to the relevant rule or skill). Do
   not change anything.

6. **Journal.** The `--log` option in step 3 already wrote
   one audit-trail line to each host whose probe ran. The
   helper prints `log failed on <host>` for any host where
   it could not; mention those in the report.

7. **No policy memory updates.** Apart from what onboarding
   itself records (`Last connected:`, a changed OS version),
   the audit is a snapshot; it does not own server state. If
   the audit uncovers a memory file that contradicts the
   live config, mention it in the "Drift detected" section
   so the user can decide what to fix.

## References

Read on demand:

- `references/probes.md` — the exact commands to run per
  category (UA, sshd, firewall, MTA, time, auto-reboot).
- `references/output-format.md` — table layout and the
  "Drift detected" section format.

## Scope and limits

- **Several hosts at once:** the audit reaches only hosts
  `bin/heinzel-fanout` accepts, so blacklisted, not yet
  onboarded and moved hosts appear as skipped, not
  audited (`rules/multi-host.md`).
- Linux (Debian family) is fully covered. RHEL/SUSE
  probes share the same shape but use `dnf`/`firewalld`/
  `zypper` equivalents. macOS hosts are skipped with a
  "macOS not yet supported" note — covering them is a
  separate effort.
- The audit does not check that running services are
  healthy (that is housekeeping's job). It only compares
  declared policy.
- BatchMode SSH means no password prompts. Hosts that need
  a passphrase get skipped — fix the agent setup
  separately.
