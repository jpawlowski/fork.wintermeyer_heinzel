# Network Profile

What heinzel records about a host's network, how
it probes it, and what counts as a finding. The
profile answers four questions before anyone
touches the network:

1. **Who owns the configuration?** Which manager,
   which file, and whether something regenerates it.
2. **What is the stack?** IPv4 and IPv6, which
   address ranges, static or dynamic.
3. **Does it work?** Default routes, name
   resolution, and outbound reachability per
   address family. Configured IPv6 is often broken
   IPv6.
4. **Does the outside agree?** A, AAAA and PTR
   records against the addresses the host really
   has.

Every probe here is read-only.

## When

- **First connection**, and on a known host whose
  directory has no `network.md` yet: run the full
  profile and write `network.md`. Announce it in one
  line ("network profile for this host — one
  moment").
- **Every later connection:** compare the uplink
  addresses (`ip -br addr show dev <uplink>`, or
  `ifconfig <uplink>` on BSD and macOS) with
  `network.md`. Re-run the full profile when they
  differ, when `Probed:` is older than 90 days, or
  when the user asks about the network.
- **After heinzel changes** anything in the network,
  DNS or firewall configuration: re-run the affected
  sections and update `network.md` in the same step.

Bundle each OS's probes into as few SSH calls as
possible (`rules/ssh-connections.md`). None of them
needs root, except where noted.

## Where it goes

`memory/servers/<hostname>/network.md`, next to
`memory.md`. `memory.md` keeps its `- IP:` line
(`rules/dns-aliases.md` depends on it) and gains one
summary line:

```markdown
- Network: dual-stack, v6 egress OK — see network.md
```

`network.md` holds current facts only. Replace
stale values; do not append history (that is what
`changelog.log` is for):

```markdown
# Network — host.example.com
Probed: 2026-09-19

## Summary
- Stack: dual-stack (v4 public, v6 GUA)
- Egress: v4 OK, v6 OK (deb.debian.org)

## Management
- Manager: netplan → systemd-networkd
- Source: /etc/netplan/50-cloud-init.yaml
- cloud-init: owns network config (datasource
  as reported by cloud-id)
- Conflicts: none

## Interfaces
- Uplink: eth0, MTU 1500, physical
- Overlay: wg0 · Containers: docker0 + 2 br-*
- Virtualization: kvm

## IPv4
- 203.0.113.10/32 static, public
- Default: via 172.31.1.1 dev eth0

## IPv6
- 2001:db8:1:2::1/64 static, GUA
- Default: via fe80::1 dev eth0, static
- RA: networkd (userspace), kernel accept_ra 0
- Forwarding: 0 · Temporary addrs: none
- Interface ID: manual · DNS64: none

## DNS
- resolv.conf: symlink → systemd-resolved stub
- Upstream: 2 v4 + 1 v6 on eth0, link-provided
- Search: none · DNSSEC: no · DoT: no
- nsswitch hosts: files dns
- Own name: hostname -f resolves to itself

## Public DNS (from workstation)
- A: matches · AAAA: matches
- PTR v4: host.example.com, forward-confirmed
- PTR v6: missing

## Findings
- INFO: no PTR for 2001:db8:1:2::1
```

Record only what a probe showed. Never infer a
hosting provider from an address range or a
gateway; name one only when `cloud-id` or the user
said so (see CLAUDE.md → Never fabricate server
facts).

**Secrets.** Netplan YAML, NetworkManager keyfiles
and `.netdev` files can hold WireGuard private keys,
Wi-Fi passphrases and 802.1X credentials. Never
print such a file whole. Grep for the keys you need,
as the probes below do. Proxy URLs can carry a
password: report that a proxy is set, never its
value. See `rules/secrets.md`.

## Probes — Linux

### A. Who manages the network

```bash
for u in systemd-networkd NetworkManager networking \
         network wicked systemd-resolved resolvconf \
         dhcpcd connman; do
  printf '%s=%s\n' "$u" \
    "$(systemctl is-active "$u" 2>/dev/null)"
done
ls -d /etc/netplan/*.yaml /etc/network/interfaces \
  /etc/network/interfaces.d/* /etc/systemd/network/* \
  /etc/sysconfig/network-scripts/ifcfg-* \
  /etc/sysconfig/network/ifcfg-* 2>/dev/null
grep -hE '^[[:space:]]*(auto|allow-hotplug|iface) ' \
  /etc/network/interfaces \
  /etc/network/interfaces.d/* 2>/dev/null
command -v networkctl >/dev/null 2>&1 \
  && networkctl list --no-pager --no-legend \
     | grep -vE ' (veth|cali|cni|flannel|vnet)'
command -v nmcli >/dev/null 2>&1 \
  && nmcli -t -f DEVICE,TYPE,STATE,CONNECTION device
if command -v cloud-init >/dev/null 2>&1; then
  echo "cloud-id=$(cloud-id 2>/dev/null)"
  grep -rlsE 'config:[[:space:]]*disabled' \
    /etc/cloud/cloud.cfg /etc/cloud/cloud.cfg.d/
fi
systemd-detect-virt 2>/dev/null
```

Then, for the uplink only (the interface of the
default route, section B):

```bash
networkctl status <uplink> --no-pager 2>/dev/null
nmcli -g GENERAL.CONNECTION device show <uplink> \
  2>/dev/null
nmcli -g ipv4.method,ipv6.method,ipv6.addr-gen-mode \
  connection show "<connection>" 2>/dev/null
nmcli -g ipv6.ip6-privacy \
  connection show "<connection>" 2>/dev/null
```

Netplan files are mode 0600 on current releases.
Read the renderer and the dynamic-addressing keys
with root or `sudo -n`, never the whole file:

```bash
grep -hE -e '^[[:space:]]*(renderer|dhcp4|dhcp6):' \
  -e '^[[:space:]]*(accept-ra|ipv6-privacy|link-local):' \
  /etc/netplan/*.yaml
```

How to read it:

- `networkctl status` names the `.network` file in
  use. A path under `/run/systemd/network/` with
  `netplan` in its name means netplan generated it:
  the source of truth is the YAML, not that file.
- `networking=active` alone is not ifupdown in
  charge. Debian runs the unit even when
  `/etc/network/interfaces` configures only `lo`.
  The `iface` lines decide.
- `network=active` is the legacy initscripts
  service (RHEL 8 and older). `wicked` is SUSE's
  manager.
- **cloud-init.** When cloud-init is installed and no
  file disables its network config, cloud-init owns
  the network. It writes the configuration always on
  the first boot of a new instance and, for some
  datasources, on every boot. Hand edits to the
  files it renders can vanish.
- **Conflict:** two managers that both show the same
  interface as managed (networkctl `configured` and
  nmcli `connected`, or an ifupdown `iface` stanza
  plus either). Record it under Conflicts.

### B. Interfaces, addresses, routes

```bash
ip -br link | grep -vE '^(veth|cali|cni|flannel|vnet)'
ip -br link | grep -cE '^(veth|cali|cni|flannel|vnet)'
for t in bond bridge vlan wireguard vxlan macvlan; do
  printf '%s: ' "$t"
  ip -o link show type "$t" 2>/dev/null \
    | cut -d: -f2 | tr -d ' ' | tr '\n' ' '
  echo
done
ip -o -4 addr show | grep -vE ': (veth|cali|cni)'
ip -o -6 addr show | grep -vE ': (veth|cali|cni)'
ip -4 route show default
ip -6 route show default
ip -4 rule; ip -6 rule
```

- The **uplink** is the device of the default route.
  Several default routes in one family are a finding
  unless their metrics differ on purpose.
- `/sys/class/net/<if>/device` exists for physical
  NICs. `docker0` and `br-*` are container bridges;
  `wg*`, `tailscale0`, `zt*` and `tun*` are overlays.
  Count container interfaces, do not list them.
- **Dynamic addresses** carry `dynamic` and a finite
  `valid_lft`: in practice DHCP for IPv4, SLAAC or
  DHCPv6 for IPv6. `proto kernel_ra` marks an
  address the kernel built from a Router
  Advertisement. A /128 with `dynamic` is usually
  DHCPv6. Confirm with the manager's own view
  (section A) before writing it down.
- `temporary` marks an RFC 4941 privacy address,
  `mngtmpaddr` the address it is derived from,
  `deprecated` an address that is no longer
  preferred.
- **Interface ID from the MAC (EUI-64):** the last 64
  bits contain `ff:fe` in the middle and match the
  link's MAC with the seventh bit flipped.
- `ip rule` beyond the defaults (local, main and
  default for IPv4; local and main for IPv6) is
  policy routing. WireGuard (`wg-quick`)
  and Tailscale add their own rules; name the owner.
- A default route with `proto ra` and `expires`
  comes from a Router Advertisement and lives only
  as long as RAs keep arriving.

### C. Kernel IPv6 settings

```bash
cat /proc/sys/net/ipv4/ip_forward
cd /proc/sys/net/ipv6/conf 2>/dev/null \
  && grep -H . */disable_ipv6 */accept_ra \
       */forwarding */use_tempaddr */addr_gen_mode \
     | grep -vE '^(veth|cali|cni|flannel|vnet)'
```

- `disable_ipv6=1` on `all` or the uplink: IPv6 is
  off.
- `accept_ra`: `0` the kernel ignores RAs, `1` it
  accepts them unless forwarding is on, `2` it
  accepts them even with forwarding. Read the
  uplink's own value: `all/accept_ra` does not
  override it. ifupdown defaults to `2` for
  `inet6 auto` but to `1` for `inet6 dhcp`, so a
  DHCPv6 host that later turns on forwarding walks
  into the trap below (see interfaces(5)).
- **Who handles RAs.** systemd-networkd always sets
  the kernel's `accept_ra` to 0 and processes RAs
  itself. So `accept_ra=0` together with a
  `proto ra` default route means a userspace
  manager does the work. Only when `accept_ra` is 1
  or 2 is the kernel in charge.
- `use_tempaddr`: `0` off, `1` generated, `2`
  generated and preferred for outgoing traffic.
- `addr_gen_mode`: `0` EUI-64, `1` none, `2` stable
  privacy, `3` random. networkd and NetworkManager
  may override it; the address itself (section B)
  is the evidence.

### D. DNS

```bash
ls -l /etc/resolv.conf
grep -m3 '^#' /etc/resolv.conf
grep -E '^(nameserver|search|domain|options)' \
  /etc/resolv.conf
[ -L /etc/resolv.conf ] || lsattr /etc/resolv.conf \
  2>/dev/null
if command -v resolvectl >/dev/null 2>&1; then
  resolvectl dns 2>/dev/null
  resolvectl domain 2>/dev/null
  resolvectl status --no-pager 2>/dev/null \
    | grep -E 'resolv.conf mode|Protocols|Current DNS' \
    | sort -u
fi
grep '^hosts:' /etc/nsswitch.conf
ss -lnu 'sport = :53'; ss -lnt 'sport = :53'
hostname; hostname -f
getent hosts "$(hostname -f)"
getent ahostsv6 ipv4only.arpa | grep -v '^::ffff:'
```

`/etc/resolv.conf` tells you who writes it:

- Symlink to `stub-resolv.conf`: systemd-resolved,
  applications ask the stub on 127.0.0.53.
- Symlink to `/run/systemd/resolve/resolv.conf`:
  resolved writes the upstream servers directly.
- Symlink to `/run/NetworkManager/…`, or header
  `Generated by NetworkManager`: NetworkManager.
- Symlink to `/run/resolvconf/…` or a resolvconf
  header: resolvconf or openresolv.
- Symlink to `/run/netconfig/…`: SUSE netconfig.
- A plain file without a generator header: static,
  maintained by hand or by cloud-init. An `i` in
  `lsattr` means someone made it immutable to stop
  a manager from rewriting it; record that.

More:

- `resolvectl dns` lists servers per link. Servers on
  a link that the manager's config does not set came
  from DHCP or from RDNSS in a Router Advertisement.
  Write "link-provided" unless the config shows the
  source.
- `ss` shows a local resolver on port 53 (resolved on
  127.0.0.53/54, unbound, dnsmasq, bind).
- `ipv4only.arpa` has only A records (RFC 7050), so
  an IPv6 answer for it comes from a DNS64 resolver
  and carries the NAT64 prefix. glibc's `getent
  ahostsv6` also lists IPv4-mapped addresses
  (`::ffff:…`) for names without AAAA; they are not
  DNS answers, hence the `grep -v`.
- `getent hosts "$(hostname -f)"` answering
  `127.0.1.1` comes from `/etc/hosts`: Debian's
  default, not a finding. It matters only for a
  service that must announce its public name (an
  MTA, for instance).

### E. Outbound reachability

See [Egress test](#egress-test).

## Probes — FreeBSD

```bash
sysrc -a | grep -E \
  -e '^(ifconfig_|ipv6_|defaultrouter|rtsold|gateway_enable)' \
  -e '^(resolv|local_unbound|dhclient|cloudinit|nuageinit)'
ifconfig -a | grep -E '^[a-z]|inet6? |nd6 options|status:'
netstat -rn -f inet | grep '^default'
netstat -rn -f inet6 | grep '^default'
sysctl net.inet.ip.forwarding net.inet6.ip6.forwarding \
  net.inet6.ip6.accept_rtadv net.inet6.ip6.use_tempaddr
grep -E '^(nameserver|search|domain|options)' \
  /etc/resolv.conf
grep -v '^#' /etc/resolvconf.conf 2>/dev/null
hostname; host "$(hostname)" 2>/dev/null
```

- `rc.conf` is the source of truth.
  `ifconfig_<if>="DHCP"` (or `SYNCDHCP`) is DHCP;
  `ifconfig_<if>_ipv6="inet6 accept_rtadv"` together
  with `rtsold_enable="YES"` is SLAAC.
- `nd6 options` on each interface: `ACCEPT_RTADV`
  accepts RAs, `IFDISABLED` means IPv6 is off.
- With `ip6.forwarding=1` FreeBSD ignores RAs by
  default. Check `sysctl -d net.inet6.ip6.rfc6204w3`
  on the host before relying on that knob.

## Probes — macOS

```bash
networksetup -listnetworkserviceorder
scutil --nwi
ifconfig | grep -E '^[a-z]|inet6? '
route -n get default 2>/dev/null \
  | grep -E 'gateway|interface'
route -n get -inet6 default 2>/dev/null \
  | grep -E 'gateway|interface'
scutil --dns | grep -E '^resolver|nameserver|search domain|if_index' \
  | head -40
sysctl net.inet.ip.forwarding net.inet6.ip6.forwarding
```

Then `networksetup -getinfo "<service>"` for the
primary service (first in `scutil --nwi`): it shows
DHCP or manual configuration for IPv4 and
Automatic, Manual or Off for IPv6.

- `ifconfig` flags on IPv6 addresses: `autoconf`
  (SLAAC), `temporary` (privacy address), `secured`
  (stable, not from the MAC).
- `scutil --dns` shows the resolver order, including
  per-domain resolvers set by VPN clients.

## Egress test

One request per address family to a host the server
already talks to, so the test adds no new third
party:

- **Debian/Ubuntu, RHEL, SUSE:** the first repository
  host in the package sources.
- **FreeBSD:** the `pkg` repository
  (`pkg -vv | grep url`).
- **macOS:** Apple's software update host
  (`swscan.apple.com`).

A `Egress test target:` line in `memory/network.md`
(fleet-wide) or in the host's `network.md` overrides
the default, e.g. for hosts behind an internal
mirror.

Linux:

```bash
grep -rhoE 'https?://[^/ "]+' /etc/apt/sources.list \
  /etc/apt/sources.list.d/ /etc/yum.repos.d/ \
  /etc/zypp/repos.d/ 2>/dev/null | sort -u | head -5
env | grep -ciE '^(https?|all)_proxy='
apt-config dump 2>/dev/null \
  | grep -ciE '^Acquire::https?::Proxy '
```

With `t` set to the chosen URL (scheme and host):

```bash
h=${t#*://}
getent ahostsv4 "$h" | head -1
getent ahostsv6 "$h" | grep -v '^::ffff:' | head -1
for f in 4 6; do
  if command -v curl >/dev/null 2>&1; then
    c=$(curl -"$f" -sS -o /dev/null -m 5 \
      -w '%{http_code}' "$t/" 2>/dev/null)
  elif command -v wget >/dev/null 2>&1; then
    wget -"$f" -q -T 5 --spider "$t/" 2>/dev/null
    c="wget-exit=$?"
  else
    c=no-client
  fi
  echo "egress$f=$c"
done
```

On FreeBSD use `fetch -4` / `fetch -6` with
`-q -T 5 -o /dev/null`; on macOS `curl` as above.

Reading the result:

- **Any HTTP status** (even 403 or 404), or wget
  exit 0 or 8: the family reaches the internet.
  curl `000`, wget exit 4, or a fetch error: it
  does not.
- **The target has no AAAA** (empty `ahostsv6`
  line): the v6 result is `inconclusive`, not `broken`. Pick
  another host from the list or ask the user for a
  target.
- **Both families fail** on one target: try the next
  host. A dead repository is not dead egress.
- **A proxy is configured** (count above 0): direct
  egress may be blocked on purpose. Record
  `Egress: via proxy` and do not report a failed
  direct test as a finding.
- ULA-only IPv6 without NAT66 has no global v6
  egress by design. Record it; it is not broken.

Finding out the public address behind NAT needs an
external echo service. Do that only when the user
asks.

## Public DNS view

Run on the **workstation**, not on the server: it
shows what clients see. Only for public addresses.

```bash
dig +short A <hostname>
dig +short AAAA <hostname>
dig +short -x <each public address>
dig +short A <name from PTR>      # forward-confirm
dig +short AAAA <name from PTR>
```

Without `dig`, use `host`. Compare with the
addresses from section B:

- An A or AAAA pointing at an address the host does
  not have breaks inbound connections for clients
  of that family. Exception: a private IPv4 on the
  host with a public A record is 1:1 NAT. Record
  `NAT` rather than a mismatch.
- A PTR that does not resolve back to the same
  address (no forward confirmation) hurts mail
  delivery.
- A generic PTR from the provider's pool (e.g.
  `dynamic-…pool.<isp>`) is not the host's own name.
  For a mail host it counts as a missing PTR.
- A GUA without an AAAA record is normal for a host
  that only connects outwards.

## Classification

**IPv4:**

- `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`:
  private (RFC 1918).
- `100.64.0.0/10`: shared address space, CGNAT
  (RFC 6598). Tailscale uses it on `tailscale0` too:
  there it is an overlay, not the uplink.
- `169.254.0.0/16`: link-local. As the only IPv4
  address it means DHCP failed.
- `192.0.0.0/29` on a CLAT interface: 464XLAT.
- `192.0.2.0/24`, `198.51.100.0/24`,
  `203.0.113.0/24`: documentation ranges, never on
  a real host.
- Anything else outside loopback and multicast:
  public.

**IPv6:**

- `2000::/3`: global unicast (GUA). Exceptions below.
- `fd00::/8`: unique local (ULA). `fc00::/8` is
  undefined; treat it as a misconfiguration.
- `fe80::/10`: link-local. Only this means IPv6 has
  no usable address.
- `fec0::/10`: site-local, deprecated since 2004.
- `2002::/16` (6to4) and `2001::/32` (Teredo):
  legacy tunnels.
- `2001:db8::/32`: documentation, never on a real
  host.
- `64:ff9b::/96`: NAT64 well-known prefix, seen in
  DNS64 answers and routes, not as a host address.

**Stack** (from the uplink's addresses, default
routes and the egress test; container bridges do
not count — Docker with IPv6 puts a ULA on its
bridge on any host):

- `dual-stack`: usable IPv4 and IPv6 GUA, both with
  a default route.
- `v4-only`: IPv6 off, or link-local only.
- `v6-only`: no IPv4 default route. Add `+ NAT64`
  when DNS64 answers, `+ 464XLAT` with a CLAT
  address.
- `v4 + ULA`: IPv6 only inside the site.
- Append `v6 broken` when a GUA and a default route
  exist but the v6 egress test fails, and `no v6
  route` when a GUA exists without a default route.

## Findings

The same list serves onboarding (the `## Findings`
section in `network.md`), housekeeping and the fleet
audit. Severities follow the housekeeping report
format.

**CRITICAL**

- Name resolution fails (`getent` on the egress
  target returns nothing in any family).
- A GUA exists and the firewall filters IPv4 but not
  IPv6. See `heinzel-security` →
  `references/firewall.md` → IPv6 coverage.

**WARN**

- `v6 broken` or `no v6 route` (see Stack). Clients
  prefer IPv6 and hang or fall back slowly; `apt`
  and `curl` stall.
- **The RA/forwarding trap:** the kernel handles RAs
  (`accept_ra=1`), forwarding is on for the uplink
  (`forwarding=1`, typically set later by Docker,
  libvirt or a VPN role), and the IPv6 default route
  comes from RAs. The kernel stops accepting RAs, and
  the route dies when its `expires` counter hits
  zero, or already has. Fix: `accept_ra=2` on the
  uplink, or a static IPv6 default route.
- Two managers claim the same interface.
- `/etc/resolv.conf` is a static file while
  systemd-resolved or NetworkManager is active: the
  manager's DNS settings are silently ignored.
- Only nameservers of a family the host cannot
  reach (e.g. IPv6 resolvers on a host with broken
  IPv6).
- A or AAAA record points at an address the host
  does not have (outside the NAT exception).
- IPv6 disabled via sysctl while an AAAA record is
  published or the manager configures IPv6.
- Several default routes in one family with equal
  metric.
- An address from a deprecated, undefined or
  documentation range (`fec0::/10`, `fc00::/8`,
  6to4, Teredo, `2001:db8::/32`, the IPv4
  documentation ranges) or `169.254.0.0/16` as the
  only IPv4 address.
- The uplink is down (`operstate` not `up`) on a
  configured interface.
- No PTR, or a PTR without forward confirmation, on
  a public address of a host that sends mail
  (`memory.md` lists an MTA).

**INFO**

- cloud-init owns the network configuration: edits
  belong in `/etc/cloud/cloud.cfg.d/`, or cloud-init's
  network config must be disabled first.
- Temporary (privacy) IPv6 addresses on a server:
  the outgoing source address rotates and breaks
  allow-lists on the far side.
- Interface ID derived from the MAC (EUI-64): the
  address changes when the NIC does and exposes the
  MAC.
- A server whose address depends on a DHCP lease.
- `/etc/resolv.conf` marked immutable.
- Only one upstream nameserver.
- No PTR on a public address (without an MTA).
- `hostname -f` does not return an FQDN, or does not
  resolve.

## Before changing the network

Network and firewall changes can cut off SSH
(CLAUDE.md → Firewall & network): discuss first.
Then:

- **Edit the owner's source, not its output.** The
  netplan YAML, not the generated `.network` file;
  the NetworkManager connection (`nmcli connection
  modify`), not `/etc/resolv.conf`; the cloud-init
  config when cloud-init owns the network.
- **Test with a safety net.** `netplan try` rolls
  back unless confirmed. For other managers,
  schedule a rollback before applying (e.g. a
  `systemd-run --on-active=5min` job that restores
  the backup and re-applies it) and cancel it once
  a fresh SSH login still works.
- Back up every file first (`rules/backups.md`) and
  re-run the profile afterwards.
