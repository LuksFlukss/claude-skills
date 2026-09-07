---
name: termius
description: Reconnect Tailscale without letting it hijack routes/DNS (`tailscale up --accept-routes=false --accept-dns=false`). Use when Tailscale being up is breaking connectivity to local bots/services (e.g. Big Pickle and other Herdr agents) because it took over the default route or DNS resolver.
---

# /termius — Tailscale up, without route/DNS takeover

Tailscale's defaults can silently break connectivity to non-tailnet services:
it may push routes for local/LAN traffic through the tailnet interface, and
MagicDNS overrides `/etc/resolv.conf` to point at `100.100.100.100`. If bots
like Big Pickle or other local Herdr agents lose connectivity as soon as
Tailscale connects, this is almost always why.

## Command

```bash
tailscale up --accept-routes=false --accept-dns=false
```

This brings/keeps Tailscale up but tells it not to accept subnet routes or
override DNS, so normal LAN/local traffic and name resolution keep working
alongside the tailnet.

## Steps

1. Run the command above.
2. Confirm Tailscale is connected: `tailscale status`.
3. If the user reports bots (Big Pickle, etc.) still can't connect, check
   `ip route` and `/etc/resolv.conf` for lingering Tailscale entries — a prior
   `tailscale up` without these flags may have left routes/DNS in place until
   the daemon restarts or `tailscale down && tailscale up ...` is re-run.

## Notes

- This does not disable Tailscale — it only stops it from taking over routing
  and DNS. The tailnet itself (SSH, direct node-to-node access) still works.
- Safe to run any time Tailscale is already up; it just re-applies the flags.

---

## Usage as a command

When invoked as `/termius`, run the command above and report whether Tailscale
came up successfully and with the expected (non-hijacking) config.
