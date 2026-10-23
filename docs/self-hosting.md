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

## Step 1 — Decide how you will sign in, then install

The **first administrator** signs in through the bundled identity provider
(Authelia) with the `admin` account the installer creates. Accounts created
with a username and password in the web client are always regular users. They
can't create the first admin.

So, **before installing**, tell the web client where its *Sign in with SSO*
button should go. In the repository's root `.env` (copy `.env.example` if it
doesn't exist yet), set:

```bash
CLIENT_OIDC_LOGIN_URL=https://api.minder.local/v1/auth/oidc/login
CLIENT_AUTHELIA_PORTAL_URL=https://authelia.minder.local
```

Replace `minder.local` with your own domain if you serve Minder under a real
one. These values are built into the web client image, which is why they go in
before the install.

Then install:

```bash
# Requires licensed access: the repository and images come with your license.
git clone <your-licensed-repository-url> minder
cd minder
bash setup.sh install --profile standard   # minimal | standard | full
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

## Step 2 — Make the web addresses work on your computer

Minder's web UIs live under `*.minder.local` names behind its own reverse proxy
and TLS certificate. Your computer needs to resolve those names and trust that
certificate:

1. **Add the hosts line** printed by the banner, for example
   `127.0.0.1 client.minder.local api.minder.local authelia.minder.local …`:
    - Linux/macOS: `/etc/hosts`
    - Windows: `C:\Windows\System32\drivers\etc\hosts` (edit as Administrator)
2. **Trust Minder's local certificate authority:**

    ```bash
    bash setup.sh trust-ca
    ```

    Then restart the browser.

**Check:** `https://client.minder.local` opens without a certificate warning.

To use Minder from other computers on your network, add the same hosts line
there with the server's LAN IP instead of `127.0.0.1`, and trust the CA there
too.

## Step 3 — Sign in as the first administrator

1. Open `https://client.minder.local` and choose **Giriş yap / Sign in** →
   **Sign in with SSO (Authelia)**.
2. Sign in as `admin` with the password from Step 1.
3. You land back in the client as an **instance admin**. The first admin to
   sign in on a fresh install also becomes **Platform Admin** (the cross-tenant
   operator role) automatically, once. See [Platform Admin](platform-admin.md)
   to grant it to others later.
4. Change the generated password in the Authelia portal
   (`https://authelia.minder.local`). See
   [Authentication](authentication.md#the-admin-password).

!!! note "No SSO button on the sign-in page?"
    Then the client was built without `CLIENT_OIDC_LOGIN_URL`. Set the two
    values from Step 1 in `.env`, rebuild the client with
    `bash setup.sh update`, and reload the page. Opening the gateway's SSO URL
    directly doesn't work: the client only accepts a sign-in that it started
    itself.

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
