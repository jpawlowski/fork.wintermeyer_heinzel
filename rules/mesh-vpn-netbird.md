# Mesh VPN: NetBird

Part of `rules/mesh-vpn.md`; read it when the probe
finds `netbird`.

- **Connected:** `Management: Connected` and `Signal:
  Connected`; `Peers count: 3/5 Connected` shows how
  many peers are reachable. `Daemon status:
  NeedsLogin`, `LoginFailed` or `SessionExpired`: not
  connected. Address: `NetBird IP`.
- **Expiry:** `Session expires`, for peers logged in
  through SSO with login expiration on.
- **Management:** the server whose dashboard holds the
  policy.
- **Keys:** the JSON state files hold the WireGuard
  private key (see NetBird SSH below for the grep).

## NetBird SSH

`netbird up --allow-server-ssh` (0.61 or newer for the
current model); on in the probe: `SSH Server:
Enabled`. Who: OIDC login (JWT) plus the access policy
in the dashboard, which reaches the host only as
hashes. Flags, root:

```bash
grep -rHE --include='*.json' \
  '"(ServerSSHAllowed|EnableSSH[A-Za-z]*|DisableSSHAuth)"' \
  /var/lib/netbird /var/db/netbird /etc/netbird 2>/dev/null
```

The active profile is the file of the `Profile:` line
(`default.json` for `default`); `/var/db/netbird` is
FreeBSD, `/etc/netbird/config.json` older clients.

- `ServerSSHAllowed: true` without anyone turning it
  on: a config from before the setting existed is
  switched on at upgrade.
- `EnableSSHRoot: true`: admits `root`.
- `DisableSSHAuth: true`: no OIDC login; any peer the
  network policy lets through gets in, with no person
  behind the login.
- SFTP and port forwarding: allowed where `true`.
- **Client side:** the NetBird client writes
  `/etc/ssh/ssh_config.d/99-netbird.conf`, a `Match`
  block for peer names with `StrictHostKeyChecking no`
  and a `ProxyCommand netbird ssh proxy` that checks
  the peer's host key against NetBird's management
  instead. `stricthostkeychecking no` there is not the
  "any host key accepted" finding.

## Who logged in

`netbird status -d` lists the open sessions (local
account, JWT user, source). Past logins, root:

```bash
grep -e 'SSH auth' -e 'SSH connection from NetBird peer' \
  /var/log/netbird/client.log
```

The second pattern is the only line without OIDC; a
`--log-file` on the service's command line wins.

## Login and changes

- **With OIDC** heinzel's login needs a browser login
  through the NetBird client; `BatchMode` cannot do it.
- `netbird down`, then `netbird up
  --allow-server-ssh=false`; the VPN is down
  meanwhile.
