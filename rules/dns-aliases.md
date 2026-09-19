# DNS Aliases

The same physical server can have multiple DNS names.
heinzel detects this automatically using the `- IP:`
field in server memory files.

**Canonical name** = the first hostname used for a
server. Additional DNS names become filesystem
symlinks to the canonical directory.

## Detection (on every new hostname)

When connecting to a hostname with no
`memory/servers/<hostname>/` directory (and not a
symlink):

1. **Resolve the IP(s).** Query A records and
   filter for addresses — a bare `dig +short` can
   return a CNAME target instead of an IP:
   ```
   dig +short A <hostname> | \
     grep -E '^[0-9.]+$'
   ```
   Verify the syntax against the dig version on
   the machine running the query (see CLAUDE.md →
   Verify Before Running). Fallback if `dig` is
   unavailable:
   ```
   python3 -c "import socket; \
     print(socket.gethostbyname('<hostname>'))"
   ```

2. **Compare against known servers.** Scan existing
   `memory/servers/*/memory.md` files (skip
   symlinks) for a matching `- IP:` line.

3. **Match found -> alias.**
   - Create symlink:
     `ln -s <canonical> memory/servers/<alias>`
   - Add `- DNS alias: <alias>` to canonical
     `memory.md`.
   - Skip OS detection.

4. **No match -> new server.** Normal first-connection
   flow. Include resolved IP as `- IP:` field.
   Once connected, record the server's own name as
   `- FQDN:`:
   ```
   hostname -f
   ```
   On Linux this is the resolver's canonical name
   for the host (so `/etc/hosts` counts); on FreeBSD
   and macOS it is the configured hostname
   unchanged. Accept it only if it contains a dot
   and is not `localhost…`. Otherwise, or if
   `hostname` is missing, use the canonical name the
   local resolver returns for the name the user
   gave:
   ```
   python3 -c "import socket; \
     print(socket.getaddrinfo('<hostname>', None, \
     flags=socket.AI_CANONNAME)[0][3])"
   ```
   Treat a name ending in `.local` like one without
   a dot: mDNS names are unique only on their own
   network segment. If neither yields a usable name,
   leave the field out rather than guess. The
   directory keeps the name it was created under;
   `- FQDN:` never renames it.

## Subsequent Connections via Alias

Follow the symlink, read canonical `memory.md`. Use
the **alias hostname** (not canonical) for SSH
commands and `user.md` lookups. Each alias can have
its own SSH user.

## IP Verification

On every connection to a known server, verify the
current IP matches `- IP:` in memory. Compare
against the full set of resolved IPs: round-robin
DNS gives a host multiple A records, and any
overlap with the stored IP(s) counts as a match.
Note multi-A hosts in server memory instead of
alarming. Only when there is no overlap at all,
**stop and tell the user.** Ask whether the server
migrated (update IP) or the alias now points
elsewhere (detach it).

## Short Names Matching More Than One Server

When the user names a host without a dot, scan
`memory/servers/*/memory.md` (skip symlinks) for
`- FQDN:` lines whose first label equals that name,
ignoring case. If more than one server matches, do
not guess and do not connect: list the matching
FQDNs and ask which server is meant, then continue
with that server's directory. One match or none:
continue as usual.

## Removing an Alias

1. Delete the symlink from `memory/servers/`.
2. Remove the `- DNS alias:` line from canonical
   `memory.md`.
3. Remove the alias from `memory/user.md` if present.
