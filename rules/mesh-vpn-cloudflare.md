# Mesh VPN: Cloudflare Tunnel

Part of `rules/mesh-vpn.md`; read it when the probe
finds `cloudflared`.

`cloudflared` connects outbound to Cloudflare and
publishes local services; no interface, no VPN
address.

- **Connected:** no status command; its log says.
- **Config:** `/etc/cloudflared`, `~/.cloudflared` or
  `/usr/local/etc/cloudflared`; a tunnel managed from
  the dashboard has its routes there, not on the host.
- **Ways in:** an ingress `service: ssh://…` publishes
  sshd; Cloudflare Access (browser SSH, short-lived
  certificates) decides who reaches it. The host
  firewall does not see it: the connection is
  outbound.

  ```bash
  grep -Hn 'ssh://' /etc/cloudflared/*.y*ml \
    /usr/local/etc/cloudflared/*.y*ml \
    ~/.cloudflared/*.y*ml 2>/dev/null
  ```

- **Keys:** `cert.pem`, the tunnel's `<UUID>.json`,
  and the token. A token count above 0 in the probe:
  the token is in `ps` for every account → move it
  into a token file (`rules/secrets.md` → Never Pass
  Secrets on the Command Line).
