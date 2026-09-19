# Mesh VPNs and Tunnels

Servers spread over sites, clouds and home offices are
often connected by a mesh VPN rather than a company
network: admins reach them through it, and they reach
each other. Per host heinzel records which VPN or
tunnel it is in, with which address and role, whether
it is connected, and whether the agent brings its own
SSH server or exposes sshd. How heinzel itself reaches
the host: `rules/access-path.md`.

Common in self-hosting, homelabs and small companies:

| Agent | Policy lives | SSH |
|---|---|---|
| Tailscale (+ Headscale) | control server | own server |
| NetBird | management server | own server |
| WireGuard (wg-quick, wg-easy, Netmaker) | the host | — |
| ZeroTier | network controller | — |
| Nebula (+ Defined Networking) | the host | admin console |
| Pangolin (Newt) | Pangolin server | own server |
| Cloudflare Tunnel | Cloudflare / config | exposes sshd |

OpenVPN and IPsec are usually site-to-site or
hub-and-spoke; record them like WireGuard.

## When to check

- **First connection:** the probe below, in the OS
  detection call (`rules/first-connection.md`).
- Before work on SSH access or accounts, before
  firewall changes on port 22 or a VPN interface, and
  before touching an agent.
- When heinzel's own login behaves unlike sshd (see
  "heinzel's own login").

## Probe (no root)

```bash
PS=$(ps ax -o comm=)
printf '%s\n' "$PS" \
  | grep -E -e '(^|/)(tailscaled|headscale|netbird)$' \
    -e '(^|/)(zerotier-one|nebula|dnclient|newt|cloudflared)$' \
    -e '(^|/)(wireguard-go|netclient|openvpn|charon(-systemd)?)$' \
  | sort -u
case $PS in *cloudflared*)
  ps ax -o comm=,args= | grep -c -- '^[^ ]*cloudflared .*--token' ;;
esac
if command -v ip >/dev/null 2>&1; then
  ip -br link show type wireguard
  ip -br addr | grep -E '^(tailscale|zt|nebula|tun)'
else
  ifconfig -g wg 2>/dev/null
  ifconfig -l
fi
ls -d /Applications/Tailscale.app \
  /Applications/WireGuard.app 2>/dev/null
if command -v tailscale >/dev/null 2>&1; then
  echo "== tailscale"
  tailscale version 2>&1 | head -n 1
  tailscale ip 2>&1
  tailscale status --json --peers=false 2>&1 \
    | grep -e '"BackendState"' -e '"Online"' \
      -e '"KeyExpiry"' -e '"Expired"'
  P=$(tailscale debug prefs 2>&1)
  printf '%s\n' "$P" | grep -e '"RunSSH"' \
    -e '"ControlURL"' -e '"OperatorUser"'
  printf '%s\n' "$P" | sed -n \
    '/"AdvertiseRoutes": null/p; /"AdvertiseRoutes": \[/,/]/p'
fi
if command -v netbird >/dev/null 2>&1; then
  echo "== netbird"
  netbird status 2>&1 | grep -e '^Daemon' \
    -e '^Management' -e '^Signal' -e '^NetBird IP' \
    -e '^Profile' -e '^SSH Server' -e '^Session expires' \
    -e '^Peers count' -e 'daemon'
fi
```

What finds what:

- **Processes:** Tailscale, a Headscale server,
  NetBird, ZeroTier, Nebula and `dnclient`, Newt,
  `cloudflared`, userspace WireGuard, Netmaker's
  `netclient`, OpenVPN, strongSwan (`charon`) — also
  inside containers. The `cloudflared` line counts
  whether its token is in `ps` arguments without
  printing them; it is anchored on the process name so
  that it does not count the probe itself.
- **Interfaces:** kernel WireGuard has no process;
  `ip -br link show type wireguard` lists it whatever
  its name (wg-quick, Netmaker's `netmaker`, NetBird's
  `wt0`), `ifconfig -g wg` on FreeBSD. On macOS names
  say little (`utun*`).
- **macOS apps:** the App Store and standalone
  Tailscale and the WireGuard app run under other
  process names; the `ls` finds them.

Not found by this probe:

- An agent in a container with its own network
  namespace (the wg-easy server, Tailscale or NetBird
  in Docker): the process shows, its interface and
  CLI do not. When that is the case, name the
  container, with root:

  ```bash
  for rt in docker podman; do
    command -v "$rt" >/dev/null 2>&1 || continue
    "$rt" ps --format '{{.Names}} {{.Image}}' 2>&1 \
      | grep -iE -e 'tailscale|headscale|netbird|wg-easy' \
        -e 'wireguard|zerotier|nebula|newt|cloudflared'
  done
  ```

  Record agent and container; its state stays
  unchecked. Rootless Podman containers show only for
  their own account.
- Services whose process names heinzel does not know
  (Firezone gateway, Twingate, others). A VPN the user
  names: record it anyway.

## Agents

Per agent: whether it is **connected** (installed and
running is not enough), when its login **expires**,
and where its keys are (`rules/secrets.md`: read only
the fields named here).

An expiry on a server is usually a forgotten setting
(servers should not expire): name it. Within 7 days,
tell the user; severities: `heinzel-housekeeping` →
Mesh VPNs.

What cuts a host off a VPN: stopping, restarting or
upgrading the agent, taking it down, a policy, ACL or
firewall change on the VPN, an expiry. Where heinzel
or other hosts depend on that path (subnet router,
WireGuard hub, lighthouse, control server):
`rules/access-path.md` → Only one way in.

Per agent, the details are in their own file:

- `rules/mesh-vpn-tailscale.md` — Tailscale and
  Headscale, Tailscale SSH
- `rules/mesh-vpn-netbird.md` — NetBird, NetBird SSH
- `rules/mesh-vpn-wireguard.md` — WireGuard, wg-easy,
  Netmaker, Firezone
- `rules/mesh-vpn-zerotier.md` — ZeroTier
- `rules/mesh-vpn-nebula.md` — Nebula, Defined
  Networking
- `rules/mesh-vpn-pangolin.md` — Pangolin's Newt
- `rules/mesh-vpn-cloudflare.md` — Cloudflare Tunnel

## SSH servers in agents

Tailscale, NetBird and Newt serve SSH themselves, on
port 22 of the VPN address. sshd does not see these
logins, so none of this applies to them:
`sshd_config` and its hardening, `authorized_keys`,
sshd's CA trust and principals, fail2ban, the sshd log,
`last`. Who may log in, and as which account, is
decided by a policy **outside the host**.

- **Firewall:** Tailscale SSH and Newt are served
  inside the agent and never pass the host firewall;
  closing 22 in ufw or firewalld does not stop them.
  NetBird puts its redirect (22 → 22022) into
  nftables/iptables itself. None shows up on port 22
  in `ss` or `lsof`; NetBird shows `:22022` on its own
  IP (kernel mode only).
- **Accounts:** Tailscale and NetBird map a person to
  a local account that must already exist; Newt
  creates them.
- **Access path:** a session that came in on an agent
  address (`rules/access-path.md`) went to the agent
  when the server port is `22022` (NetBird) or the
  address is Tailscale's with `RunSSH` on; otherwise
  to sshd.

### Where the policy is

heinzel sees the compiled Tailscale rules (root) and
the NetBird and Newt switches on the host, never the
policy that admits people: the Tailscale admin
console, Headscale's policy, the NetBird dashboard,
the Pangolin server. Ask the user for it, or for read
access, when a finding depends on it. Never change it:
that is a change for every host at once, made by the
user.

### Who logged in

In each agent's file. These logins are missing from
`last`, the sshd log and the activity check's sshd
lines.

### heinzel's own login

When heinzel came in on an agent's SSH server, then:

- keys, certificates and `sshd_config` play no part;
  a `Permission denied` means the policy, not the key.
- The agent's own conditions (Tailscale check mode,
  NetBird OIDC) are in its file.
- To reach sshd, use the host's other address.

### Changing it

Turning an agent's SSH server on or off opens or
closes a way in: ask first, and keep another way in
(`rules/access-path.md` → Only one way in).

The commands per agent are in its file.

## Security audit

For `heinzel-security`: put the probes into the audit's
sshd calls. One report line per agent found; none when
there is none.

- An agent, or its SSH server, on while `network.md` →
  Mesh VPN has no entry for it or says off → **WARN**
  (a way in nobody recorded)
- Tailscale, per rule, root counting via `"root":
  "root"` or `"*": "="` without `"root": ""`: root by
  `accept` (no check) → **WARN**, for `any: true`
  principals → **CRITICAL**; other accounts for
  `any: true` → **WARN**
- NetBird: `EnableSSHRoot` and `DisableSSHAuth` both
  `true` → **CRITICAL** (root for any peer the network
  policy admits, no person behind it); either one alone
  → **WARN**
- Tailscale `OperatorUser` set → **INFO**, name the
  account (it can turn SSH on without root)
- SFTP or port forwarding allowed (NetBird flags,
  Tailscale `allowLocalPortForwarding`) → **INFO**
- Session recording (`recorders`) → OK, name it
- Policy not readable from the host (Tailscale without
  root, NetBird always) → **INFO** "policy not
  checked", and ask the user for it
- Newt SSH on → **INFO**, and list the accounts and
  `/etc/sudoers.d/90-pangolin-*` files it created
- Nebula `sshd.enabled` → **INFO**, name `listen` and
  the `authorized_users`
- Cloudflare Tunnel ingress `ssh://` → **INFO** (sshd
  reachable through Cloudflare; Access decides who); its
  token in `ps` arguments → **WARN**
- Agent found, no SSH server of its own → OK
- Expiry and connection state: `heinzel-housekeeping`
  → Mesh VPNs and WireGuard

Report line: `heinzel-security` →
`references/report-format.md`.

## Memory

Per server, the details go into a `## Mesh VPN`
section of `memory/servers/<hostname>/network.md`, the
host's network profile where there is one; otherwise
create the file with this section only. One line per
agent, also when its SSH server is off, so turning it
on later shows as a change:

```markdown
## Mesh VPN
- Tailscale 1.102.4, 100.101.102.103, connected,
  control Headscale 0.29.3 hs.example.com, no key
  expiry, routes 10.0.0.0/24; SSH on, root by check,
  others accept (netmap 2026-09-19)
- WireGuard wg0 10.8.0.5, hub vpn.example.com
- Tailscale in container ts-proxy
  (tailscale/tailscale); state unchecked
- Nebula nebula1 192.168.100.7, groups servers, cert
  to 2027-01-10; admin sshd off
- NetBird 0.79.0, 100.92.1.7, not connected
  (SessionExpired); SSH on, flags unchecked
```

Without root, mark what is missing `unchecked`.

`memory.md` gets one summary line, which the rules and
skills that read only `memory.md` key on:

```markdown
- Mesh VPN: Tailscale (SSH on), WireGuard — see
  network.md
```

Fleet-wide, in `memory/network.md`, one entry per
network: agent, control server (Tailscale, or
Headscale with host and version; ZeroTier controller;
Nebula lighthouses; WireGuard hub), where the policy
lives, subnet routers and what they route. Whether the
workstation is in it goes to `memory/user.md`
(personal).
