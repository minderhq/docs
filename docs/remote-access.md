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

- **No real DNS + TLS** — by default (`MINDER_TLS_MODE=local-ca`) the reverse
  proxy serves certificates from a local CA that only machines you have set up
  trust (`bash setup.sh trust-ca` installs it on the host). Other devices get a
  browser warning, and nothing validates who you're actually connecting to over
  the open internet.
- **Admin password** — the SSO (Authelia) admin password is auto-generated per
  deployment and printed once during setup. Make sure you actually recorded it
  before exposing the login page to the internet; if you didn't, rotate it first.
- **IP-whitelist assumptions** — internal admin dashboards (reverse-proxy,
  message-queue, and graph-database UIs) are protected by an IP-whitelist that
  assumes a trusted network. A public IP breaks that assumption entirely.

## Recommended: a private VPN

A WireGuard-based mesh VPN (or any VPN you already run) puts your phone or
laptop on the same private network as the machine running Minder, with no ports
opened to the public internet at all. If the Minder host is already a member of
such a network, you already have everything you need.

1. **Join the same VPN** on the device you want to reach Minder from, with the
   same account or network as the Minder host.
2. **Find the Minder host's VPN address.** Look it up on the Minder host itself
   (the interface of your VPN client). It is usually a private address such as
   `100.x.y.z` or `10.x.y.z`.
3. **Add the same hosts-file entry you use for LAN access, but with the VPN
   address** instead of `127.0.0.1`:

   ```text
   100.x.y.z chat.minder.local
   ```

4. Open `https://chat.minder.local` (or the raw VPN address) exactly as you would
   on the LAN — same certificate warning (until the device trusts the local CA), same login.

!!! tip "On a phone"
    Most mobile VPN apps don't let you edit `/etc/hosts` directly. Browse to the
    VPN address directly (e.g. `https://100.x.y.z`), or use a local
    DNS/hosts-editing app.

!!! note "VPN-provided DNS names"
    If your VPN gives hosts resolvable names, you can use one of those as the
    hosts-file target, or skip the hosts-file step and browse straight to that
    name. Some services key off the `chat.minder.local` hostname for CORS/routing,
    so if anything looks broken, prefer the hosts-file approach with the exact
    name.

## What's not supported yet

Public internet exposure with real DNS and a CA-issued certificate isn't set up
out of the box. The VPN approach above is the supported path today — not a
workaround for a first-class feature that exists elsewhere.
