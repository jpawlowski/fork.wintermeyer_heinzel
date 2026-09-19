# Firewall

## Linux

Verify a firewall is installed, active, and the default incoming
policy is deny/drop. An inactive or missing firewall on a Linux
server is **CRITICAL** — aligned with the housekeeping severity.

Three variants count as a firewall: ufw, firewalld, and native
nftables (Debian's own default: `nftables.service` loading
`/etc/nftables.conf`). Check them in that order and judge the
host by the first one that is active. Report **CRITICAL** "No
active firewall" only when none of the three is.

### Debian/Ubuntu (ufw)

```bash
ufw status verbose
```

- Not installed or inactive → check native nftables below
- Active but default incoming is not `deny` → **WARN** "Firewall
  default incoming policy is not deny"
- Active and default deny → OK

### RHEL/Fedora/SUSE (firewalld)

```bash
firewall-cmd --state
firewall-cmd --get-default-zone
```

Then check the default zone's target:

```bash
firewall-cmd --zone=<zone> --get-target
```

- Not running → check native nftables below
- Zone target is `ACCEPT` → **WARN** "Default zone target is
  ACCEPT (allows all incoming)"
- Zone target is `default` (reject/drop) → OK

### Native nftables

Needs root or `sudo -n`; `nft` refuses to list for a normal
user.

```bash
systemctl is-active nftables
nft list chains | grep -E '^table|chain |hook input'
```

A chain whose `type` line carries `hook input` filters incoming
traffic. A packet has to pass every input chain of its family,
so one chain that drops is enough — an `accept` policy in
another table does not undo it. Default deny means one of:

- the input chain has `policy drop;`, or
- it has `policy accept;` but its last rule is an
  unconditional `drop` or `reject`. Check with:

  ```bash
  nft list chain <family> <table> <chain> | tail -3
  ```

Results:

- `nftables.service` inactive and no input chain → nothing
  filters (**CRITICAL** above, if ufw and firewalld are
  inactive too)
- Input chain, but neither condition holds → **WARN**
  "nftables input policy is not deny"
- Input chain that drops, but `nftables.service` is inactive
  and neither ufw nor firewalld runs → **WARN** "nftables
  rules will not survive a reboot"
  (`systemctl is-enabled nftables netfilter-persistent`
  shows whether anything reloads them)
- Default deny and `nftables.service` active → OK

A table of family `inet` filters IPv4 and IPv6; `ip` filters
only IPv4.

### Mixed frameworks

On a host where `iptables` writes to nf_tables, rules loaded
through `iptables-legacy` still filter packets, but `nft` and
`iptables` do not show them. Check which framework `iptables`
uses and whether legacy tables exist:

```bash
iptables -V
cat /proc/net/ip_tables_names /proc/net/ip6_tables_names
```

Read the proc files first: `iptables-legacy` loads its kernel
modules on demand, so calling it on a clean host creates the
tables it is meant to look for. Only when `iptables -V` says
`(nf_tables)` and a proc file lists a table:

```bash
iptables-legacy -S | grep -vc '^-P'
ip6tables-legacy -S | grep -vc '^-P'
```

`iptables -S` on such a host also prints `# Warning:
iptables-legacy tables present`.

- Either count > 0 → **WARN** "iptables-legacy rules active
  next to nf_tables, invisible to nft". List them with
  `iptables-legacy -S` and report which tool loads them.
- Both 0, or no legacy table → OK

### Docker published ports

Docker rewrites the destination of published ports in the
`nat` table, before packets reach the INPUT chain that ufw
and firewalld filter
(https://docs.docker.com/engine/network/packet-filtering-firewalls/).
`-p 8080:80` is reachable from the internet even behind
`ufw default deny incoming`. Check whenever `docker` is
installed (`command -v docker`); `docker ps` needs root or
the `docker` group:

```bash
docker ps --format '{{.Names}} {{.Ports}}'
```

A port is public when its entry starts with `0.0.0.0:`,
`[::]:` or `:::` (for example `0.0.0.0:8080->80/tcp`).
Entries bound to `127.0.0.1:` or `[::1]:` are local only;
entries without `->` are not published. A specific public
address is as exposed as `0.0.0.0`.

A public port counts as restricted only by rules Docker
evaluates before its own:

```bash
iptables -S DOCKER-USER
ip6tables -S DOCKER-USER
```

Anything beyond `-N DOCKER-USER` (older engines add a
`-j RETURN` as well) is a user rule; read it to see which
ports and sources it covers.

Docker Engine 29 added an nftables backend, still
experimental and not available in Swarm mode
(https://docs.docker.com/engine/network/firewall-nftables/).
It is on when `/etc/docker/daemon.json` sets
`"firewall-backend": "nftables"`; the running engine says
which backend it uses:

```bash
docker info --format '{{.FirewallBackend.Driver}}'
```

An engine without that field predates the backend and uses
iptables. The nftables backend has no DOCKER-USER chain;
restrictions live in a separate table with a base chain on
Docker's hooks, visible in `nft list chains`.

- Public port not covered by a DOCKER-USER rule (or its
  nftables equivalent) → **WARN** "Docker publishes
  <container> <port> past the firewall". Suggest binding it
  to `127.0.0.1` behind a reverse proxy, or a DOCKER-USER
  rule that limits the sources.
- All published ports local or restricted → OK

## macOS

Check Application Firewall status:

```bash
/usr/libexec/ApplicationFirewall/socketfilterfw \
  --getglobalstate
```

- Disabled → **INFO** (not WARN — common on macOS behind NAT,
  consistent with housekeeping severity)
- Enabled → OK
