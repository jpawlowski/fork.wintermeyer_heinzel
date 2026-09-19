# Access Control

Rules for the server blacklist and read-only server
list. Both use the same file format and lookup logic.

## Shared File Format

Plain list — one entry per line. An optional leading
`- ` bullet is allowed and ignored. Everything after
`#` is a comment; blank lines are ignored. Entries
can be a hostname (matched against the target
hostname) or an IP address (matched against the
resolved IP). IPs are the most robust form — prefer
them. Files are created on first need — do not
pre-create them.

```markdown
# Example

- server.example.com   # bullet optional
203.0.113.50
```

The user can add or remove entries by asking heinzel
to edit the file, or by editing it directly.

## Shared Lookup Logic

For both files, the check is (stop at the first
match):

1. If the file does not exist, skip (nothing to
   check).
2. Is the target hostname, or the name `ssh -G`
   maps it to (`rules/dns-aliases.md` → Detection
   step 1), listed?
3. Resolve the target's IP(s)
   (`rules/dns-aliases.md` → Detection step 1).
   Is a resolved IP listed?
4. Resolve each listed hostname to its IP(s) the
   same way, once per session, and compare against
   the target's IP(s). This catches DNS aliases of
   listed hosts that a plain string match would
   miss.

If nothing resolves, fall back to exact string
matching and tell the user explicitly that the
IP-level check could not be performed. Err on the
side of caution for anything ambiguous.

## Server Blacklist

**File:** `memory/blacklist.md`

**When to check:** before every connection attempt —
before OS detection, before DNS alias detection,
before any SSH command. This is the very first step
when a user mentions a server.

**On match:** refuse to connect. Tell the user:
"This server is blacklisted in
`memory/blacklist.md`. I will not connect to it."
Do not proceed. Do not ask for override. Do not run
any SSH commands against the server.

## Read-Only Servers

**File:** `memory/readonly.md`

**When to check:** right after the blacklist check,
before OS detection and DNS alias detection.

**On match:** announce read-only mode to the user:
"This server is marked read-only in
`memory/readonly.md`. I will connect and inspect,
but I will not make any changes."
Proceed with the connection — unlike the blacklist,
read-only does not block access.

**Allowed in read-only mode:**
- All read-only operations: SSH inspection, status
  commands, reading files and logs
- Housekeeping checks and security audits
- Local memory and changelog updates, including
  creating `todo.md` (a purely local file — only
  the remote host is read-only)
- `logger -t heinzel` entries on the server

**Blocked in read-only mode:**
- Package install, update, or remove
- Service start, stop, enable, disable, or restart
- Config file edits on the server
- Firewall, user/group, or file write changes
- Reboots

**Deferred modifications:** when a blocked action is
needed, announce it briefly and continue with
read-only work. At session end, present a
modification report in the same format as the
unprivileged mode sysadmin report.

**No override:** read-only mode is a hard constraint.
The user must remove the entry from
`memory/readonly.md` before heinzel will modify the
server.
