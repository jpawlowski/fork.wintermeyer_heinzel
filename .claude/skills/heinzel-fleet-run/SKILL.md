---
name: heinzel-fleet-run
argument-hint: "[--all | hostname ...] <what to run or ask>"
description: Run one command or question across several known
  servers in parallel and report the answers grouped, so hosts
  that agree show once and outliers stand out. Use when the user
  names two or more servers or "all servers" for one task —
  "which kernel runs everywhere?", "check disk space on web1,
  web2 and web3", "auf allen Servern nachsehen, ob X läuft",
  "roll this config out to the web servers". Read-only questions
  and changes both; changes follow the canary and approval steps
  below. Not for one server, and not for housekeeping or a
  security audit across hosts (those run one subagent per host;
  see rules/multi-host.md → Limits).
---

# heinzel-fleet-run

One task, several servers, one call. The mechanics — the
helper, what it skips, the guard, onboarding — are in
`rules/multi-host.md`. Read it first; this skill is the
workflow around it.

## Workflow

1. **Targets.** The hosts the user named, or `--all` for
   "all servers". Hosts without `memory/servers/<host>/` get
   their first connection on their own first
   (`rules/first-connection.md`), one at a time, before the
   fan-out.

2. **Onboard** all targets in one call and act on the
   output as `rules/multi-host.md` → Onboarding for a
   fan-out says:

   ```bash
   bin/heinzel-fanout --onboard web1.example.org web2.example.org
   ```

3. **Write the script.** One POSIX `sh` script for all
   targets. Verify command syntax on the target versions
   first (`CLAUDE.md` → Verify Before Running); when the
   targets differ in OS or family, branch on `uname -s` or
   `/etc/os-release` inside the script. Privileged commands
   use `sudo -n`, never interactive sudo.

4. **Read-only task:** run it.

   ```bash
   bin/heinzel-fanout --read \
     --log '[<operator> as {user}] read-only: checked disk usage' \
     web1.example.org web2.example.org <<'EOS'
   df -h /
   EOS
   ```

5. **Change:** show the target list from
   `--write --dry-run` to the user, then follow
   `rules/multi-host.md` → Changes on several hosts
   (one question naming every host, canary, stop at the
   first surprise).

6. **Report.** One line per group: hosts, then the answer.
   Majority first, outliers after. Name unreachable and
   skipped hosts on one line each. Nothing else — see
   `CLAUDE.md` → Talking to Humans.

7. **Record.** For a read-only run, `--log` already left
   the journal line. For a change, record per host as
   `rules/multi-host.md` says.
