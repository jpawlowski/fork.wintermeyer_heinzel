# Several Hosts at Once

How heinzel runs the same work on more than one
server: a fleet question ("which kernel runs
where?"), a change rolled out to a group, a fleet
audit. One server at a time stays the default; this
file applies as soon as a request names two or more.

## Where the time goes

Not in SSH. A shared connection answers in
milliseconds and the first login takes about a
second. What costs is every model turn and every
line of output that lands in the conversation.
Twenty hosts probed one call each means twenty
turns and twenty outputs, most of them identical.

So a fan-out has three jobs:

1. **One call for all hosts.** One tool call, one
   permission prompt, hosts run in parallel.
2. **Collect locally.** Each host's output goes to a
   file; identical outputs print once, with the list
   of hosts that gave them.
3. **Keep every rule.** Access control, onboarding,
   the taboo guard and the journal apply per host
   exactly as they do one host at a time.

## The helper: `bin/heinzel-fanout`

Use it for every fan-out. It holds the standard SSH
options, the host checks and the guard in one place.
`bin/heinzel-fanout --help` lists every option.

```bash
bin/heinzel-fanout --read web1.example.org web2.example.org <<'EOS'
uname -r
EOS
```

- **Pick the mode honestly.** `--read` for scripts
  that only inspect, `--write` for anything that
  changes a host. `--write` skips hosts in
  `memory/readonly.md`; `--read` reaches them.
- **Pass the script as a heredoc in the same call.**
  That way the permission prompt shows what will
  run. Never feed it from a file you wrote for the
  purpose.
- **`--all`** means every server in
  `memory/servers/`, minus aliases (symlinks) and
  example hosts.
- **`--dry-run`** prints targets and skip reasons
  without connecting. Run it first before any
  `--write` on more than a handful of hosts, and
  show the list to the user.
- **`--log`** writes the journal line on each host
  whose script exited 0, and only there. `{user}`
  becomes that host's SSH user, so the envelope from
  `rules/changelog.md` fits every host:
  `--log '[Alice as {user}] <what> — because <why>'`.

### What it refuses or skips

| Case | Result |
|---|---|
| Script or `--log` line denied by the guard | whole run refused |
| Guard missing or crashed | whole run refused |
| Host name that is not a plain DNS name | whole run refused |
| Blacklisted (name, IP, or IP of a listed name) | `blocked` |
| Read-only host with `--write` | `readonly` |
| No `memory/servers/<host>/` yet | `new` |
| Example host, only with `--all` | `example` |
| Resolved IP does not match `- IP:` in memory | `STOP` |
| No SSH user in `memory/user.md` | `nouser` |
| The local machine or `Mode: local` | `local` |

A `STOP` line means the same as on a single host
(`rules/dns-aliases.md` → IP Verification): tell the
user and ask before touching that host again.

A `new` host gets its first connection on its own,
with the SSH user interview and the memory file
from `rules/first-connection.md`. The helper never
onboards a new host.

### The guard

The Bash tool's guard sees only the command line. A
script redirected from a file never reaches it. So
the helper runs `.claude/hooks/guard-taboos.sh` over
the script and the log line itself, before it
connects to anything.

This does not widen the guard's limits either way.
A script built at runtime or fetched from elsewhere
defeats any string matcher, fanned out or not. The
rule stays: never rephrase, encode or indirect a
command to get it past the guard.

### Reading the result

```text
run: /tmp/heinzel-fanout.Xa81  mode: read  targets: 5  parallel: 8
blocked   db.example.org        blacklisted in memory/blacklist.md
unreach   web4.example.org      ssh: connect to host … Operation timed out

== 3 host(s), rc=0: web1.example.org web2.example.org web3.example.org
6.12.38+deb13-amd64

== 1 host(s), rc=0: web5.example.org
6.1.0-37-amd64
```

- Groups are sorted by size: the majority first,
  outliers below it.
- Long outputs are cut at `--max-lines` (40); the
  full text stays in the run directory
  (`out/<host>`, `rc/<host>`). Read it from there
  instead of re-running.
- `unreach` means the script never started there
  (connection, login or socket failed). Handle it
  per `rules/ssh-unreachable.md` for that host
  alone; never retry the whole fan-out. A script that
  started and then broke off shows up as a group
  with its rc and output, so a half-applied
  `--write` is never mistaken for an untouched host.
- Exit status: 0 all targets succeeded, 1 at least
  one failed or was unreachable, 2 refused.

## Onboarding for a fan-out

The pipeline in `rules/first-connection.md` applies
to every host in a fan-out. There is no fleet
exception. It runs in two calls:

1. **Local steps, per host:** blacklist, read-only
   and IP check happen inside the helper. Read each
   target's `memory.md`, `changelog.log` and
   `todo.md` yourself.
2. **Remote steps, all hosts at once:**

   ```bash
   bin/heinzel-fanout --onboard --all
   ```

   Prints OS and version, the remote user and the
   heinzel journal of the last 7 days per host.
   Compare the OS line with memory and update it
   where it changed, set `Last connected:`, and
   summarise the activity as
   `rules/activity-check.md` says. A host whose
   `###activity-rc=` is not 0 had no working
   activity check: say so.

This call also opens the shared SSH connection to
every host. Every later call in the next 10 minutes
reuses it, so there is no separate warm-up step.

For `--write`, onboarding is always its own call
before the change, never combined with it: recent
activity by someone else is a reason to stop.

## Changes on several hosts

Everything that needs a question on one host needs
it on many. A service restart, a firewall change or
a reboot still asks once and names every host. The
answer covers exactly those hosts and that run.

- **Canary first.** Run a `--write` on one host,
  check the result, then the rest.
- **Back up per host** as `rules/backups.md` says,
  inside the same script, before the edit.
- **Stop on the first surprise.** When a group shows
  an outcome you did not expect, stop and report it.
  Do not continue with the remaining hosts.
- **Record per host.** Server memory and
  `changelog.log` are updated for each host the
  change reached, from its own output.

## Limits

- **Parallelism.** `-P 8` by default. Lower it when
  many hosts sit behind one jump host, one NAT or
  one IPS: that device counts all of them together
  (`rules/ssh-connections.md`).
- **Key agents that confirm each use** ask once per
  host, all at once at the start.
- **Heavy work per host** — housekeeping, a security
  audit, anything that reads a lot and decides per
  host — belongs to one subagent per host instead.
  Each works in its own context and returns only its
  report. Hand each subagent the host, the skill,
  and this file.
- **Different commands per host, or copying files:**
  not the helper's job. Run those per host with the
  standard options.
- **macOS and FreeBSD targets** work. The scripts you
  send must fit each OS: `uname -s` inside the
  script, or separate runs per OS.
