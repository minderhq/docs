# Remote access

By default, Minder is reachable only from the machine it's installed on — or from
other devices on the same LAN once you add a hosts-file entry (see
[Self-hosting](self-hosting.md)). This guide covers reaching your instance from
**outside** your home network: a phone, a laptop on another network, and so on.

## Don't port-forward to the public internet

!!! danger "Do not expose Minder's reverse proxy directly to the internet"
    A default deployment is **not** hardened for direct internet exposure.
    Port-forwarding `443`/`8000`/etc. on your router is the wrong tool for
    "I want to use this from outside my house." A VPN gives you the same result
    without any of the exposure below.

- **No real DNS + TLS** — the reverse proxy serves a self-signed certificate.
  Browsers warn on every visit, and nothing validates who you're actually
  connecting to over the open internet.
- **Admin password** — the SSO (Authelia) admin password is auto-generated per
  deployment and printed once during setup. Make sure you actually recorded it
  before exposing the login page to the internet; if you didn't, rotate it first.
- **IP-whitelist assumptions** — internal admin dashboards (reverse-proxy,
  message-queue, and graph-database UIs) are protected by an IP-whitelist that
  assumes a trusted network. A public IP breaks that assumption entirely.

## Recommended: Tailscale

[Tailscale](https://tailscale.com) — or any WireGuard-based VPN — puts your
phone or laptop on the same private network as the machine running Minder, with
no ports opened to the public internet at all. This isn't hypothetical setup
work: if the Minder host is already a tailnet member (check with `tailscale
status` on that machine), you already have everything you need.

1. **Join the same tailnet** on the device you want to reach Minder from:
   install Tailscale, run `tailscale up`, and sign in with the same
   account/tailnet as the Minder host.
2. **Find the Minder host's tailnet IP.** Run `tailscale status` on the Minder
   host itself and note the `100.x.y.z` address next to its hostname.
3. **Add the same hosts-file entry you use for LAN access, but with the tailnet
   IP** instead of `127.0.0.1`:

   ```text
   100.x.y.z chat.minder.local
   ```

4. Open `https://chat.minder.local` (or the raw tailnet IP) exactly as you would
   on the LAN — same self-signed-certificate warning, same login.

!!! tip "On a phone"
    Most Tailscale mobile apps don't let you edit `/etc/hosts` directly. Browse
    to the tailnet IP directly (e.g. `https://100.x.y.z`), or use a local
    DNS/hosts-editing app.

!!! note "MagicDNS"
    If your tailnet has **MagicDNS** enabled, `tailscale status` shows resolvable
    `<hostname>.<tailnet>.ts.net` names instead of raw IPs. You can use one of
    those as the hosts-file target, or skip the hosts-file step entirely and
    browse straight to `https://<hostname>.<tailnet>.ts.net`. Some services key
    off the `chat.minder.local` hostname for CORS/routing, so if anything looks
    broken, prefer the hosts-file approach with the exact name.

## What's not supported yet

Public internet exposure with real DNS and a CA-issued certificate isn't set up
out of the box. The tailnet approach above is the supported path today — not a
workaround for a first-class feature that exists elsewhere.
