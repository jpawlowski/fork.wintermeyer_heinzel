# Firewall

## Linux

Verify a firewall is installed, active, and the default incoming
policy is deny/drop. An inactive or missing firewall on a Linux
server is **CRITICAL** — aligned with the housekeeping severity.

### Debian/Ubuntu (ufw)

```bash
ufw status verbose
```

- Not installed or inactive → **CRITICAL** "No active firewall"
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

- Not running → **CRITICAL** "No active firewall"
- Zone target is `ACCEPT` → **WARN** "Default zone target is
  ACCEPT (allows all incoming)"
- Zone target is `default` (reject/drop) → OK

### IPv6 coverage

A firewall that filters only IPv4 leaves every
service open over IPv6. Check this whenever the host
has a global IPv6 address (`ip -6 addr show scope
global`, or `network.md`).

**ufw:**

```bash
grep '^IPV6=' /etc/default/ufw
```

`IPV6=no` → ufw writes no IPv6 rules.

**firewalld:** filters both families in one ruleset.
No extra check.

**Neither ufw nor firewalld:** first find out which
kernel framework the `iptables` command writes to:

```bash
iptables -V
```

- `(nf_tables)`, or no `iptables` at all: nftables
  holds every rule, including those added through
  the `iptables` command. `iptables -S` shows only
  the latter and misses native rules from
  `/etc/nftables.conf`, so read nftables directly:

  ```bash
  nft list chains | grep -E '^table|hook input'
  ```

  IPv6 is filtered when a chain with `hook input`
  sits in a table of family `inet` or `ip6` and has
  `policy drop` (or ends in a drop/reject rule — then
  check its rules with `nft list chain <family>
  <table> <chain>`). Input chains only in family `ip`
  are the gap.
- `(legacy)`: the old framework, invisible to `nft`.
  Compare the policies of both families:

  ```bash
  iptables -S INPUT | head -1
  ip6tables -S INPUT | head -1
  ```

  `-P INPUT DROP` for IPv4 next to `-P INPUT ACCEPT`
  for IPv6 is the gap.

No input chain in any family means no inbound
filtering at all — report that under "No active
firewall" above, not here.

- The gap above → **CRITICAL** "Firewall does not
  filter IPv6"

## macOS

Check Application Firewall status:

```bash
/usr/libexec/ApplicationFirewall/socketfilterfw \
  --getglobalstate
```

- Disabled → **INFO** (not WARN — common on macOS behind NAT,
  consistent with housekeeping severity)
- Enabled → OK
