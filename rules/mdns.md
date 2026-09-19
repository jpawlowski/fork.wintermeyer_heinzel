# mDNS and .local Names

Names ending in `.local` are link-local (RFC 6762).
Multicast DNS (mDNS) resolves them on the local
network, a DNS server may answer them too, and the
two can disagree.

## Which Source Answered

On first contact with a `.local` name, or when its
`memory.md` has no `- Resolved via:` line, compare
the two sources:

- **Unicast DNS:** the `dig` query from
  `rules/dns-aliases.md` → Detection step 1; run
  it if this connection has not. `dig` never uses
  mDNS.
- **mDNS on macOS** (always on, via mDNSResponder):
  `dns-sd -G v4 <name> & sleep 2; kill $!`. A
  non-zero `IF` column means the link answered.
- **mDNS on Linux** (only with systemd-resolved mDNS
  or an `mdns` module on the `hosts:` line of
  `/etc/nsswitch.conf`):
  `resolvectl query -4 -p mdns <name>`, else
  `avahi-resolve -4 -n <name>`. Without either,
  `ssh` never uses mDNS here, even if
  `avahi-resolve` answers: the source is DNS.

Record the result in `memory.md` as
`- Resolved via: mDNS`, `DNS`, or `DNS and mDNS`.

## DNS and mDNS Disagree

If both sources answer with different addresses,
**stop and tell the user**: name, both addresses,
which source gave which. Do not pick one, and do not
treat the difference as a DNS alias or a migration.
Ask which host is meant.

Common causes:

- A unicast DNS zone named `.local`, typical of
  older Active Directory setups. RFC 6762
  Appendix G advises against it.
- A name conflict on the link: the second host to
  claim the name renames itself (`<host>-2.local`),
  so `<host>.local` may be another machine than the
  one in memory.

## Responder on the Server

Once per server, when `memory.md` has no
`- mDNS responder:` line, check in the first call
after OS detection whether the server announces
itself:
who listens on UDP 5353, per
`rules/port-check.md` → Check Commands (on macOS
`lsof -nP -iUDP:5353`; mDNSResponder always runs).
On Linux add `systemctl is-active avahi-daemon` to
name the process without root.

Record it as `- mDNS responder: <name>` or `none`.
Changing it follows `rules/service-reload.md` and
CLAUDE.md → Firewall Awareness for Service Changes.

Do not browse the network for mDNS services
(`avahi-browse`, `dns-sd -B`).
