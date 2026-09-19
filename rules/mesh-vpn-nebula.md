# Mesh VPN: Nebula

Part of `rules/mesh-vpn.md`; read it when the probe
finds `nebula` or `dnclient`.

Config usually `/etc/nebula/config.yml` (the service
passes `-config`); Defined Networking's managed client
is `dnclient` with its state in `/var/lib/defined`.
With root (`nebula-cert` may be missing where only
`nebula` is installed; then ask for the dates):

```bash
nebula-cert print -path /etc/nebula/host.crt
sed -n '/^lighthouse:/,/^[a-z]/p; /^sshd:/,/^[a-z]/p' \
  /etc/nebula/config.yml
```

- **Connected:** no status command; the logs say
  whether handshakes succeed.
- **Expiry:** `notAfter` of the host certificate, and
  of `ca.crt`. Expired: the host drops out.
- **Certificate:** name, networks (own address),
  groups. **Lighthouses:** `am_lighthouse`, `hosts`.
- **Policy:** `firewall.inbound` and
  `firewall.outbound` in the config, by Nebula group.
- **Keys:** `pki.key` (`host.key`), `sshd.host_key`;
  print only the blocks above.
- **Admin console:** `sshd.enabled` (off by default)
  starts an SSH console on `sshd.listen` (never port
  22) for `authorized_users`. It controls Nebula
  itself, it is no login shell; when on, record who
  may use it.
