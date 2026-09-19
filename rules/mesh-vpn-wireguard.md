# Mesh VPN: WireGuard

Part of `rules/mesh-vpn.md`; read it when the probe
finds a WireGuard interface or `wireguard-go`.

Plain `wg-quick`, systemd-networkd, NetworkManager,
and tools built on it (wg-easy, Netmaker, Firezone).
With root:

```bash
wg show
```

- `wg show` prints peers, endpoints, allowed IPs and
  the last handshake, and hides the keys. Other `wg`
  subcommands and output modes may print them: use
  only plain `wg show`.
- **Connected:** WireGuard has no such state;
  `latest handshake` says when traffic last flowed.
  Without `persistent keepalive`, an old handshake
  only means no recent traffic. No expiry.
- **Keys:** `/etc/wireguard/*.conf` (`PrivateKey`),
  systemd-networkd `*.netdev`, NetworkManager
  connections. Own address: `ip addr`.
- **Policy:** each peer's `AllowedIPs` and the host
  firewall; Netmaker and Firezone keep theirs on their
  server.
- **Hub:** a peer every host has, with a fixed
  endpoint.
