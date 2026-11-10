# Self-hosting Minder

Minder's core is a **proprietary product** — local-first, running entirely on
hardware you own and provisioned by a single command. Self-hosting the core needs
**licensed access** (the repository and container images are private); a freemium
trial is planned.

> The **open-source** side of Minder — the [plugin SDK](plugins/index.md), the
> plugin catalog, and the web client — is freely available. This page covers
> running the (licensed) core; the steps below assume you have access.

This page takes you from an empty machine to your first answer, in order. Do
the steps in sequence; each one ends with a check.

## Requirements

- **Docker** with Compose. On Windows and macOS that means Docker Desktop.
- **Python 3**. It drives the `setup.sh` provisioning CLI.
- **Hardware.** A Raspberry Pi 4 (arm64) is a validated minimum, with small
  models. For comfortable local AI, aim for 16 GB+ RAM. An NVIDIA GPU makes
  answers several times faster. [Connect your models](model-providers.md#sizing-will-the-model-fit)
  explains what fits.
- **Windows:** Docker runs inside a WSL 2 VM. Cap its memory so Windows keeps
  enough for the browser and for a native Ollama. In `%UserProfile%\.wslconfig`:

    ```ini
    [wsl2]
    memory=16GB          # about half the RAM of a 32 GB machine

    [experimental]
    autoMemoryReclaim=dropCache
    ```

    Run `wsl --shutdown` once afterwards for it to take effect.

## Choosing a domain and TLS mode

Minder is served from **one hostname**, `MINDER_DOMAIN`, with path routing:

| Path | What |
|---|---|
| `/` | the web console |
| `/api/…` | the API gateway |
| `/auth/…` | sign-in (Authelia: SSO, OIDC, the self-service portal) |
| `/c/<slug>`, `/embed.js` | public chatbot page and its embed widget: `<script src="https://<MINDER_DOMAIN>/embed.js" data-bot="<slug>" async></script>` |

Because everything shares one origin, the browser makes no cross-origin calls,
and you need **one DNS record and one certificate**. Sign-in, OIDC redirects,
emailed links and CORS all derive from `MINDER_DOMAIN`. You don't set them one
by one.

Pick the domain and who terminates TLS for it:

| `--tls-mode` | Use it when | What happens |
|---|---|---|
| `local` (default) | trying Minder on one machine or a LAN | Minder's own CA signs a certificate for the domain; you trust the CA once per device (`bash setup.sh trust-ca`). The default domain `minder.local` needs a hosts-file line. |
| `proxy` | you already run a TLS-terminating reverse proxy with a valid certificate (Caddy is the tested one) | Minder serves plain HTTP on port 80 and accepts it **only** from the proxy's own address (`--trusted-proxies`, a single address such as `192.168.1.10/32`, never a range). It trusts `X-Forwarded-*` only from there, so cookies stay `Secure` and links stay `https`. The proxy must overwrite `X-Forwarded-Host`, `-Proto` and `-For`, and strip `X-Forwarded-Uri`. The operator UIs on `*.minder.local` are not routed in this mode: reach them over the loopback ports or an SSH tunnel. |
| `acme` | the domain has public DNS pointing at this host and port 443 is reachable | Let's Encrypt (TLS-ALPN-01) issues the certificate. |

Set them at install time (below) or later:

```bash
bash setup.sh domain minder.example.com --tls-mode proxy --trusted-proxies 192.168.1.10/32
bash setup.sh restart
bash setup.sh domain        # shows the current settings and URLs
```

### Example: behind Caddy, reached over a tailnet

A common home setup has Caddy on one machine holding a wildcard certificate for
`*.home.example.com`, and Minder on another machine. Devices reach both over a
Tailscale tailnet, and split DNS (for example an AdGuard or Pi-hole rewrite)
resolves `minder.home.example.com` to Caddy.

1. Install Minder in proxy mode, trusting only Caddy's own address: the one
   Minder sees connections from (its LAN address, or its single tailnet
   address such as `100.101.102.103/32`). Never a range: every listed address
   may forge `X-Forwarded-*`. Optionally bind port 80 to the interface Caddy
   uses with `MINDER_HTTP_BIND=<minder-lan-address>` in `.env`.

    ```bash
    bash setup.sh install --profile standard \
      --domain minder.home.example.com --tls-mode proxy \
      --trusted-proxies 192.168.1.10/32
    ```

2. Add a site to Caddy. With no `trusted_proxies` of its own, Caddy
   overwrites `X-Forwarded-For`, `-Proto` and `-Host` with the real values;
   the `header_up -X-Forwarded-Uri` line drops a client-supplied one:

    ```caddyfile
    minder.home.example.com {
        tls /etc/caddy/certs/wildcard.crt /etc/caddy/certs/wildcard.key
        reverse_proxy http://<minder-host>:80 {
            header_up -X-Forwarded-Uri
        }
    }
    ```

3. Point `minder.home.example.com` at Caddy in your DNS (the split-DNS rewrite).
   Phones and laptops on the tailnet then open `https://minder.home.example.com`
   with no hosts-file edits and no certificate warnings.

!!! warning "Keep port 80 closed to everything but the proxy"
    In proxy mode the Minder host serves plain HTTP on port 80. Minder refuses
    every other source address, but firewall it to the proxy as well.

## Step 1 — Install

The **first administrator** signs in through the bundled identity provider
(Authelia) with the `admin` account the installer creates. Accounts created
with a username and password in the web client are always regular users. They
can't create the first admin. The web client's *Sign in with SSO* button needs
no configuration: it points at `/api` on the console's own domain.

```bash
# Requires licensed access: the repository and images come with your license.
git clone <your-licensed-repository-url> minder
cd minder
bash setup.sh install --profile standard   # minimal | standard | full
# Your own domain (see above): add --domain <name> [--tls-mode proxy|acme]
```

- `--profile` picks the initial **bundles** (capability groups) to enable.
- The installer asks one question, **who can join**: `invite` (default), `open`
  or `sso_only`. See [Organizations & teams](organizations-teams.md).
- Add `--model-tier auto` to choose models for your hardware now. You can also
  do it later; see [Step 4](#step-4-choose-your-models).

!!! warning "Write down the admin password"
    During install the output shows a box **🔑 Authelia Admin Password
    (generated)** with `Username: admin` and a password. It is shown **once**.
    It is also stored in `.env` as `MINDER_AUTHELIA_ADMIN_PASSWORD`.

**Check:** the install ends with a *Ready!* banner listing the URLs.

## Step 2 — Make the web address work on your computer

With your own domain and DNS (proxy or acme mode), skip to the check at the end
of this step. With the default `minder.local` and its local CA, your computer
needs to resolve the name and trust the certificate:

1. **Add the hosts line** printed by the banner, for example
   `127.0.0.1 minder.local chat.minder.local …`:
    - Linux/macOS: `/etc/hosts`
    - Windows: `C:\Windows\System32\drivers\etc\hosts` (edit as Administrator)
2. **Trust Minder's local certificate authority:**

    ```bash
    bash setup.sh trust-ca
    ```

    Then restart the browser.

**Check:** `https://<MINDER_DOMAIN>` (for example `https://minder.local`) opens
without a certificate warning.

To use Minder from other computers on your network, add the same hosts line
there with the server's LAN IP instead of `127.0.0.1`, and trust the CA there
too.

## Step 3 — Sign in as the first administrator

1. Open `https://<MINDER_DOMAIN>` and choose **Giriş yap / Sign in** →
   **Sign in with SSO (Authelia)**.
2. Sign in as `admin` with the password from Step 1.
3. You land back in the client as an **instance admin**. The first admin to
   sign in on a fresh install also becomes **Platform Admin** (the cross-tenant
   operator role) automatically, once. See [Platform Admin](platform-admin.md)
   to grant it to others later.
4. Change the generated password in the Authelia portal
   (`https://<MINDER_DOMAIN>/auth/`). See
   [Authentication](authentication.md#the-admin-password).

!!! note "No SSO button on the sign-in page?"
    Then `.env` sets `CLIENT_OIDC_LOGIN_URL` empty. Set it to
    `/api/v1/auth/oidc/login`, rebuild the client with `bash setup.sh update`,
    and reload the page. Opening the gateway's SSO URL directly doesn't work:
    the client only accepts a sign-in that it started itself.

**Check:** the user menu shows `admin`, and the sidebar has *Kuruluş /
Organization* and *Sağlayıcılar ve varsayılanlar / Providers & defaults*.

## Step 4 — Choose your models

Minder works out of the box with a small bundled model. Most people want to
pick models that suit their hardware, or use one they already run:

- **Keep the bundled Ollama** and choose a hardware tier:
  `bash setup.sh model-tier` (suggest), then
  `bash setup.sh model-tier apply <tier>` and `bash setup.sh restart`.
- **Use your own Ollama**, vLLM, LM Studio, or a cloud provider (OpenAI,
  Anthropic, OpenRouter, z.ai). See **[Connect your models](model-providers.md)**.
  It covers each option step by step, including what must reach what, sizing,
  and privacy.

Decide on the **embedding model before uploading many documents**. A
knowledge base's embedding model can't be changed later.

**Check:** *Yapay zekâ ve sistem → Modeller / Models* lists your models, and
*Providers & defaults* shows the defaults you expect.

## Step 5 — Verify the whole path

```bash
bash setup.sh verify --deep
```

This runs as a dedicated synthetic test organization (`minder-synthetic`,
created on the first run and reused afterwards). It uploads sample documents,
asks questions, and checks the answers and their citations, the local-only
guard and tenant isolation. It then deletes the knowledge bases and pipelines
it made. It takes a few minutes. Every failing check prints a fix hint.

**Check:** the summary shows `fail: 0`.

## Step 6 — Your first knowledge base and question

Follow the [Quickstart](quickstart.md): create a knowledge base, upload
documents, create a pipeline and ask. The client's *Ask* page then answers with
numbered sources.

## Step 7 — Bring in your team

Invite people from *Kuruluş → Kullanıcılar / Organization → Users*, or set up
domain auto-join or SSO; see [Organizations & teams](organizations-teams.md).
Email is off by default. For invitations by email and self-service password
reset, configure an SMTP relay: [Transactional email](email.md).

## Capability bundles

You turn capabilities on and off as units, not by editing Compose files. `core`
is the always-on kernel; `rag`, `graph-rag`, `inference`, `chat`, `voice` and
`monitoring` layer on top. See [Capability bundles](bundles.md) for the commands
and how services are claimed.

## Check it is up

Host ports are **loopback-bound** by default (reachable on the box, not the LAN);
external access is via the reverse proxy, SSO-gated.

```bash
bash setup.sh status                     # every service and its health
docker ps -a --filter health=unhealthy   # should be empty
curl http://localhost:8000/health         # api-gateway, the one host-published API
bash setup.sh verify                      # fast read-only checks
```

Other services' ports are published only with `MINDER_DEV_PORTS=1`.

## Update

```bash
git pull
bash setup.sh update      # rebuild + rolling restart
bash setup.sh verify --deep
```

See [Upgrading](upgrades.md) for the full procedure, checks and rollback.

## Manage it

Everything that isn't chat has a modern control-plane UI (the `minder-client`
SPA) — see [Using Minder](using-minder.md) for a tour; chat itself is
OpenWebUI. Extend the platform with [plugins](plugins/index.md).

!!! note
    Deeper operations, architecture, and hardening guides ship with the licensed
    core; the plugin-ecosystem docs on this site are the openly available subset.
