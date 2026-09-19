# OS Detection (mandatory first step)

Before doing any work on a server, you **must** know
its OS.

## On first connection

0. **Check access control and DNS alias.** For remote
   servers: check blacklist, then read-only list
   (see `rules/access-control.md`), then DNS aliases
   (see `rules/dns-aliases.md`). If the hostname is
   an alias for a known server, skip OS detection.

1. Determine the OS, the login shell and the
   architecture:
   ```
   ssh … <host> 'uname -s; ps -o comm= -p $$; uname -m'
   ```
   Send it exactly like this: single quotes, so the
   local shell does not expand `$$`; no pipes,
   redirects or `&&`; and `ps` not last, because
   bash and dash exec the last command of `-c` in
   place and `ps` would then report itself. The
   account's login shell runs it, and that is not
   always sh — OPNsense runs root's commands in
   csh, pfSense gives other users tcsh, FreeBSD
   before 14.0 gives root csh. A command passed over
   SSH skips the console menus of both firewalls.

   If the first line is anything but `Linux`,
   `FreeBSD` or `Darwin` — a menu, a banner, "This
   account is currently not available" — the
   account has no command shell. Stop and show the
   user the output. Never answer a menu over SSH:
   the same menus reboot the machine or reset it to
   factory defaults.

   The second line is the shell (compare its
   basename; macOS may print `-zsh` or a path).
   Record it per SSH user in server memory
   (`Shell: csh (root)`). Unless it is `sh`,
   `bash`, `dash`, `ash`, `ksh` or `zsh`, every
   command with sh syntax (`2>/dev/null`, `$(…)`,
   `VAR=x cmd`, `[ … ]`) goes through the `sh -s`
   bundle from `rules/ssh-connections.md` §1, for
   the whole session — the activity check and the
   sudo probe included. That also covers an error
   in place of the second line (fish rejects `$$`,
   busybox `ps` may reject `-p`): record
   `Shell: unknown` and wrap.

2. **If Linux** — detect distro and version:
   ```
   . /etc/os-release && \
     echo "${ID}|${VERSION_ID}|${PRETTY_NAME}"
   ```
   Distro families: `debian`, `rhel`, `suse`.
   Map the distro to a family via the os-release
   `ID` and `ID_LIKE` fields (e.g. `ubuntu` →
   `debian`; `centos`, `rocky`, `alma`, `fedora` →
   `rhel`; `opensuse*` variants → `suse`).
   `ID=haos`, or `ID=alpine` with a Supervisor
   (step 5), has no family: see step 5.
   If no family file matches (e.g. Arch), tell
   the user, proceed cautiously with generic
   commands, and apply extra verify-before-running
   care.
   Read `rules/<family>.md`. Gather hardware info
   (`lscpu`, `free -h`, `df -h`).

3. **If macOS** — detect version and arch:
   ```
   sw_vers -productVersion && uname -m
   ```
   Read `rules/macos.md`. Gather hardware info
   (`sysctl` for CPU/RAM, `df -h`).

4. **If FreeBSD** — detect version and arch:
   ```
   freebsd-version && uname -m
   ```
   Read `rules/freebsd.md`. Gather hardware info
   (`sysctl` for CPU/RAM, `df -h`,
   `zpool status` if ZFS).

5. **Check for a platform.** Put the markers into
   the same call as step 2 or 4:
   - Linux: `command -v pveversion ha;
     echo "${SUPERVISOR_TOKEN:+supervisor}"` (prints
     only whether the token is set, never the
     token)
   - FreeBSD: `which opnsense-version pfSense-upgrade`

   | Base    | Marker                          | Platform file       |
   | ------- | ------------------------------- | ------------------- |
   | Debian  | `pveversion`                    | `rules/proxmox.md`  |
   | FreeBSD | `opnsense-version`              | `rules/opnsense.md` |
   | FreeBSD | `pfSense-upgrade`               | `rules/pfsense.md`  |
   | none    | `ID=haos`, or `ha` + Supervisor | `rules/haos.md`     |

   On a match, read the platform file after the
   family file (Home Assistant OS has none). It
   wins wherever the two disagree. Record it in
   server memory (`Platform: …`, see
   `rules/server-memory.md`). Housekeeping,
   security audit, fleet audit and the activity
   check read the platform file's sections of the
   same name instead of the baseline:
   `## Housekeeping and Audits` and `## Logs`.

6. Create a server memory file.

## On subsequent connections

Subsequent connections run the same pipeline as the
first (see `rules/first-connection.md`), including
the blacklist and read-only checks. Specific to
known servers: read the memory file, changelog, and
`todo.md` (if present) before any work, read the
matching rule file and the platform file from
`Platform:`, and verify the OS and platform
versions in one call — update memory if they
changed. Take the shell from `Shell:` instead of
probing it again.
