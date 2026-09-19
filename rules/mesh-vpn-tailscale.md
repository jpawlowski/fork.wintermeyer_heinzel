# Mesh VPN: Tailscale and Headscale

Part of `rules/mesh-vpn.md`; read it when the probe
finds `tailscaled` or a Tailscale app.

- **Connected:** `BackendState` `Running` and
  `"Online": true` (in `Self`, reached the control
  server). `NeedsLogin`, `NeedsMachineAuth` (waits for
  approval), `Stopped` (`tailscale down`), or
  `"Expired": true`: not connected. Addresses:
  `tailscale ip`.
- **Expiry:** `KeyExpiry` (in `Self`); none on a server
  with key expiry disabled in the admin console.
- **Control server:** the same client works with
  Tailscale's hosted one or a self-hosted Headscale.
  `ControlURL` `https://controlplane.tailscale.com` or
  `https://login.tailscale.com` is Tailscale's, with
  the policy in its admin console. Anything else is
  self-hosted; check from the workstation (no SSH):

  ```bash
  curl -fsS <ControlURL>/health
  curl -fsS <ControlURL>/version
  ```

  `{"status":"pass"}` is Headscale; `/version` (0.26
  and newer) gives its version. No answer: ask the
  user what runs there.
- **Headscale:** the policy is on its server
  (`headscale policy get`, or the file named by
  `policy.path`); when that server is in
  `memory/servers/`, heinzel reads it there, with that
  host's own onboarding. SSH check mode and
  `localpart:` users need Headscale 0.29 or newer.
  While it is down, logged-in nodes keep their
  connections for a while, but new logins, key
  renewals and policy changes stop.
- **Routes:** an `AdvertiseRoutes` list makes the host
  a subnet router (`0.0.0.0/0` and `::/0`: exit node).
- **`OperatorUser`** may change the agent's settings
  without root, SSH included.

## Tailscale SSH

`tailscale set --ssh`; on in the probe:
`"RunSSH": true` (since 1.100 also `tailscale get
ssh`). Linux, and macOS with the open-source
`tailscaled` only: the App Store and standalone apps
cannot serve SSH. Who: tailnet identity plus the `ssh`
rules (accept or check) of the policy file.

The rules that apply to this node arrive compiled
with its network map (root):

```bash
T=$(printf '\t')
tailscale debug netmap 2>&1 \
  | sed -n "/^$T\"SSHPolicy\": null/p; /^$T\"SSHPolicy\": {/,/^$T}/p"
```

The format is internal to Tailscale and may change;
read it, do not build on it. Each rule has:

- `principals`: `userLogin` (a person), `nodeIP` (a
  device, also tagged ones) or `any: true` (everyone
  on the tailnet).
- `sshUsers`: requested account → local account.
  `"root": "root"` admits root; `"*": "="` admits every
  account by its own name, root included unless the
  map also has `"root": ""` (how `autogroup:nonroot`
  arrives; an empty value never matches).
- `action`: `accept: true` is accept mode;
  `holdAndDelegate` (a URL) is check mode — the person
  confirms with the identity provider again first.
  `recorders` means sessions are recorded.

`SSHPolicy: null` with `RunSSH` on: no rule admits
anyone to this node. Group names and a check rule's
`checkPeriod` stay with the control server.

## Who logged in

```bash
journalctl -u tailscaled --since "7 days ago" \
  --no-pager -q | grep 'access granted to'
```

Each line names the tailnet login and the local
account (`ssh-user "root"`).

## Login and changes

- **Check mode** prints a URL and waits until the user
  confirms in a browser. The shared connection
  (`CLAUDE.md` → SSH Options) is then reused for its
  lifetime.
- `tailscale set --ssh=false`, and a restart of
  `tailscaled`, end every Tailscale SSH session.
