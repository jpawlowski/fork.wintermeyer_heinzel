# Mesh VPN: ZeroTier

Part of `rules/mesh-vpn.md`; read it when the probe
finds `zerotier-one` or a `zt*` interface.

With root (`zerotier-cli` needs the auth token):

```bash
zerotier-cli info
zerotier-cli listnetworks
```

- **Connected:** `info` says `ONLINE` (`TUNNELED`:
  online over the TCP relay, slow), and the network's
  status in `listnetworks` is `OK`. `ACCESS_DENIED`
  (not authorized on the controller), `NOT_FOUND`,
  `REQUESTING_CONFIGURATION`,
  `AUTHENTICATION_REQUIRED`: not connected.
- **Expiry:** only with single sign-on: `AUTH OK,
  expires in: …` or `AUTH EXPIRED`.
- **Controller:** the first 10 hex digits of the
  network ID are the controller's node address. Equal
  to this node's own address, or a `controller.d`
  directory in the ZeroTier home: this host is a
  self-hosted controller. Otherwise my.zerotier.com or
  another self-hosted one (ztncui, zero-ui): ask. Flow
  rules and member authorization live only there.
- **Keys:** `identity.secret` and `authtoken.secret`
  in `/var/lib/zerotier-one` (FreeBSD
  `/var/db/zerotier-one`, macOS
  `/Library/Application Support/ZeroTier/One`).
