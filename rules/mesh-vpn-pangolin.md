# Mesh VPN: Pangolin (Newt)

Part of `rules/mesh-vpn.md`; read it when the probe
finds `newt`.

Newt connects a site to a Pangolin server (reverse
proxy and VPN); it runs as a process or container,
without its own unit. Its WireGuard is in userspace
unless started `--native`.

- **Connected:** no status command; its log says.
- **Config:** `~/.config/newt-client/config.json` of
  the account that runs it, `CONFIG_FILE`, or
  `--config-file`. `newt --show-config` masks the
  secret; the file and a `--secret` argument do not.
- **SSH:** on unless `--disable-ssh` (`DISABLE_SSH`,
  `disableSsh`), see `rules/mesh-vpn.md` → SSH servers
  in agents. Its auth daemon **creates local accounts**
  (`useradd`) and sudo rules
  (`/etc/sudoers.d/90-pangolin-<user>`; sudo skips the
  file when the name has a dot, so `jane.doe` gets no
  sudo), and writes the CA it admits to
  `/etc/ssh/ca.pem`. Pangolin, not heinzel, manages
  those accounts. With `newt` running, a
  `trustedusercakeys /etc/ssh/ca.pem` in sshd makes
  Pangolin the user CA for sshd too; the path alone
  proves nothing.

## Who logged in and changes

- Logins: Newt's own output (container or process
  log); heinzel has no probe for it.
- `--disable-ssh` and a restart turn its SSH server
  off; the tunnel is down meanwhile.
