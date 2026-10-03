# Transactional email

Minder can send a small set of **transactional emails**: self-service password
reset links, and security notices such as "your password was changed". Email is
**off by default**. This page follows the operator's path from "do I need this?"
to running it in production.

!!! info "At a glance"
    - Off by default (`EMAIL_BACKEND=none`). Without it, an install works exactly
      as it did before email existed.
    - Production backend: `smtp`, through your own relay or a mail provider.
    - Configuration lives in the root `./.env`. **Changes take effect only when
      the api-gateway container is recreated** (see
      [Apply the change](#apply-the-change)).
    - Operator tooling: `bash setup.sh email ...`. Health: the `email` object in
      the api-gateway `/health`.

## 1. Decide whether you need email

**Without email** (`EMAIL_BACKEND=none`), everything works except the features
below. Sign-in, SSO, chat, knowledge bases, plugins and administration are not
affected. Password recovery is done by an administrator:

- through the admin reset endpoints and the Users page (see
  [Authentication](authentication.md#password-management)), or
- as break-glass from the host: `bash setup.sh user reset-password <username>`.
  It prints a temporary password once and ends every session for that account.

**With email**, these turn on:

| Feature | What the user sees |
|---------|--------------------|
| Self-service password reset | A **Forgot password?** link on the sign-in page. It sends a one-time reset link to local accounts. |
| Notices for SSO accounts | An SSO-linked account that asks for a reset gets a notice that points to its identity provider. It never gets a reset link. |
| Security notices | A "your password was changed" email on **every** password path: self-service reset, change-password and administrator resets. A "your sessions were ended" email when sessions are revoked without a password change. |

Email is sent from the **api-gateway** process. A background worker drains a
database outbox, so no request ever waits on SMTP.

!!! warning "SaaS deployments: fail-fast"
    With `SAAS_MODE=true`, the gateway **refuses to start** (there is no
    override) when any of these is true:

    - `EMAIL_BACKEND` is `none`, `log` or `capture`.
    - `EMAIL_FROM` is empty.
    - `SMTP_SECURITY=none`.
    - `MINDER_CLIENT_BASE_URL` is not an `https` URL, or is still the shipped
      default.
    - `PASSWORD_RESET_EMAIL_FOR_PLATFORM_ADMINS=true` or
      `ADMIN_RESET_TEMP_PASSWORD_WITH_EMAIL=true`.

    The check covers configuration only. If the SMTP relay is unreachable, that
    shows up in readiness and never crash-loops the gateway.

## 2. Choose a backend

| `EMAIL_BACKEND` | Use it for | Behaviour |
|-----------------|------------|-----------|
| `none` (default) | Installs without email | Nothing is sent. The reset routes are not mounted (`404`), and **Forgot password?** is hidden. |
| `smtp` | **Production** | Sends through an SMTP relay or provider. |
| `log` | Local development | Writes each rendered email, **reset links included**, to the api-gateway log at INFO. The gateway logs a warning at startup. Never use it on a host with real users. |
| `capture` | Automated tests | Keeps messages in process memory, so in practice nothing arrives anywhere. |

`log` and `capture` are refused in SaaS mode.

**Picking an SMTP relay.** Any SMTP relay or transactional-mail provider works:
your organization's mail server, a cloud provider's SMTP interface, or a
dedicated transactional-email service. You need:

- a hostname and port,
- a username and password (or an API key used as the password), unless the
  relay authenticates by IP address,
- permission to send as the address you will put in `EMAIL_FROM`. Most
  providers require you to verify the sending domain first.

Choose the TLS mode to match the port:

| Port | `SMTP_SECURITY` | Meaning |
|------|-----------------|---------|
| 587 | `starttls` (default) | Plain connection upgraded with STARTTLS. If the server does not offer STARTTLS, the send **fails**. Minder never falls back to plaintext. |
| 465 | `tls` | TLS from the first byte ("implicit TLS" / SMTPS). |
| 25 (or a local relay) | `none` | **Plaintext, no certificate check.** Self-host only, and only for a relay on a trusted network (for example, one on the same host). The gateway logs a warning at startup, and SaaS mode refuses it. |

With `starttls` and `tls`, the relay's certificate is verified against the
container's system trust store and must match `SMTP_HOST`. There is no setting
for a private CA, so a relay with a self-signed certificate fails with a `tls`
error.

!!! tip
    Many cloud and residential networks block **outbound port 25**. Use 587 or
    465 to reach an external provider.

## 3. Configure

All settings go in the root `./.env`, the only file you edit (see
[Production](production.md#single-source-of-truth)). `.env.example` has a
commented **TRANSACTIONAL EMAIL** block you can uncomment.

### A complete working example

```bash
# --- Transactional email -----------------------------------------------------
EMAIL_BACKEND=smtp
EMAIL_FROM=Minder <no-reply@mail.example.com>
EMAIL_REPLY_TO=support@example.com          # optional
EMAIL_PRODUCT_NAME=Minder                   # the only branding in templates

SMTP_HOST=smtp.provider.example
SMTP_PORT=587
SMTP_SECURITY=starttls
SMTP_USERNAME=minder-relay-user
SMTP_PASSWORD=change-me                     # a secret: .env only, never commit

# Links in emails are built from this. It must be the address users open
# the client at, as they reach it.
MINDER_CLIENT_BASE_URL=https://minder.example.com

# --- Password reset (on by default once email is on) ------------------------
PASSWORD_RESET_EMAIL_ENABLED=true
PASSWORD_RESET_TOKEN_TTL_MINUTES=30
# SSO_PASSWORD_HELP_URL=https://idp.example.com/password-help
```

!!! warning "`MINDER_CLIENT_BASE_URL` matters"
    Reset links are `<MINDER_CLIENT_BASE_URL>/reset-password#token=...`, built
    only from this setting and never from a request's `Host` header. If it still
    has the default (`https://client.minder.local`), external users get links
    that do not resolve.

!!! note "`$` in passwords"
    Docker Compose interpolates `$` in `.env` values. If `SMTP_PASSWORD`
    contains `$`, write it as `$$`.

### All settings

| Key | Default | Allowed / notes |
|-----|---------|-----------------|
| `EMAIL_BACKEND` | `none` | `none`, `log`, `capture`, `smtp` |
| `EMAIL_FROM` | *(empty)* | Required for any backend other than `none`. `Name <addr@domain>` or a bare address. Its domain is the one that needs [sender DNS](#4-sender-dns-spf-dkim-dmarc). |
| `EMAIL_REPLY_TO` | *(empty)* | Optional `Reply-To` header. |
| `EMAIL_PRODUCT_NAME` | `Minder` | 1-64 characters on one line. The only branding templates render. |
| `EMAIL_DEFAULT_LOCALE` | `en` | Fallback language. The templates ship in `en` and `tr`. |
| `EMAIL_MAX_PER_MINUTE` | `60` | 1-10000. Global send ceiling. Excess mail waits in the outbox; it is not dropped. |
| `EMAIL_MAX_PER_RECIPIENT_PER_DAY` | `10` | 1-1000. Rolling 24 h per address. Security notices are exempt. |
| `EMAIL_OUTBOX_PII_RETENTION_DAYS` | `7` | 0 up to `EMAIL_OUTBOX_RETENTION_DAYS`. See [Retention](#retention). |
| `EMAIL_OUTBOX_RETENTION_DAYS` | `30` | 1-3650 |
| `SMTP_HOST` | *(empty)* | Required when `EMAIL_BACKEND=smtp`. |
| `SMTP_PORT` | `587` | 1-65535 |
| `SMTP_USERNAME` / `SMTP_PASSWORD` | *(empty)* | Login is attempted only when `SMTP_USERNAME` is set. |
| `SMTP_SECURITY` | `starttls` | `starttls`, `tls`, `none` |
| `SMTP_TIMEOUT_SECONDS` | `10` | Greater than 0 and at most 60. Applies per socket operation. |
| `PASSWORD_RESET_EMAIL_ENABLED` | `true` | Self-service reset is available when email is on **and** this is `true`. |
| `PASSWORD_RESET_TOKEN_TTL_MINUTES` | `30` | 15-60. Startup refuses any other value. |
| `PASSWORD_RESET_MAX_PER_HOUR` | `3` | Per account, at least 1. |
| `PASSWORD_RESET_MIN_INTERVAL_SECONDS` | `60` | Per account, not negative. |
| `PASSWORD_RESET_MIN_RESPONSE_MS` | `400` | 0-10000. Response-time floor for reset requests. |
| `AUTH_EMAIL_IP_LIMIT_PER_MINUTE` | *(empty)* | Per-IP limit on the reset routes. Empty means the same as the auth rate limit. `0` turns the per-IP limit off on these routes only. |
| `SSO_PASSWORD_HELP_URL` | *(empty)* | An http(s) URL, used as the link in the SSO-account notice. When empty, the notice links to the identity provider's issuer URL. |
| `EMAIL_PLACEHOLDER_DOMAINS` | *(empty)* | Comma-separated domains that are never mailed. The SSO fallback placeholder domain is always included. |
| `PASSWORD_RESET_EMAIL_FOR_PLATFORM_ADMINS` | *(unset)* | When unset, it is `false` whenever Platform Admin 2FA is required (always in SaaS), and `true` otherwise. |
| `ACCOUNT_TOKEN_RETENTION_DAYS` | `7` | At least 1. |

An inconsistent value (an out-of-range number, `smtp` without `SMTP_HOST`, or a
backend without `EMAIL_FROM`) **stops the gateway at startup** with a message
that names the setting:

```text
Refusing to start, email configuration: SMTP_HOST is required when EMAIL_BACKEND=smtp
```

### Apply the change

The backend is chosen **once, at startup**. The gateway also decides at startup
whether to mount the reset routes. Docker Compose bakes `.env` values into the
container when it **creates** it. A plain container restart therefore keeps the
old values, and you have to recreate the container:

```bash
bash setup.sh start      # re-applies ./.env and recreates only the services whose configuration changed
# or the full cycle:
bash setup.sh restart    # stop everything, then start
```

!!! warning
    `bash setup.sh restart api-gateway` and `docker compose restart api-gateway`
    restart the **existing** container with its **old** environment. They do
    not pick up `.env` edits.

Then confirm that the gateway came up with the new backend:

```bash
docker logs minder-api-gateway 2>&1 | grep -i "transactional email backend"
# Transactional email backend: smtp
```

## 4. Sender DNS (SPF, DKIM, DMARC)

Receiving providers check that mail claiming to come from your domain was
authorized by that domain. Mail that fails these checks lands in spam or is
rejected outright, and large mailbox providers require them. You publish the
records on the domain of `EMAIL_FROM` (in the example, `mail.example.com`).
Using a dedicated subdomain for transactional mail keeps its reputation separate
from your staff mailboxes.

Minder does **not** sign mail itself. DKIM signing is done by your relay or
provider, so the exact record values come from **your provider's domain-setup
page**. The shapes are:

| Record | Name | Type | Example value | Purpose |
|--------|------|------|---------------|---------|
| SPF | `mail.example.com` | TXT | `v=spf1 include:<provider-spf-domain> -all` | Lists the servers allowed to send for the domain. A self-run relay uses `ip4:<relay public IP>` instead of `include:`. Publish **one** SPF record per name. |
| DKIM | `<selector>._domainkey.mail.example.com` | TXT (or CNAME to the provider) | `v=DKIM1; k=rsa; p=<public key>` | Lets receivers verify the relay's signature. The provider gives you the selector and key. |
| DMARC | `_dmarc.mail.example.com` | TXT | `v=DMARC1; p=none; rua=mailto:dmarc-reports@example.com` | Tells receivers what to do when SPF/DKIM do not align with the From domain, and where to send reports. Start with `p=none`, then move to `quarantine` or `reject` once the reports are clean. |

If you run your own relay, also give its public IP a **reverse DNS (PTR)**
record that matches its hostname.

Check the records from any machine:

```bash
dig +short TXT mail.example.com                          # SPF: "v=spf1 ..."
dig +short TXT <selector>._domainkey.mail.example.com    # DKIM: "v=DKIM1; ..."
dig +short TXT _dmarc.mail.example.com                   # DMARC: "v=DMARC1; ..."

# Windows / no dig:
nslookup -type=TXT _dmarc.mail.example.com
```

DNS changes can take up to the record's TTL to become visible. To confirm that
receivers actually accept the mail, send a test message (next step) to:

- a mailbox at a large provider, then open the message's original headers and
  look for `Authentication-Results: ... spf=pass ... dkim=pass ... dmarc=pass`,
  or
- an online deliverability checker that gives you a one-off address (for
  example mail-tester.com). An SPF/DKIM/DMARC lookup tool such as MXToolbox can
  also validate the published records.

## 5. Verify

### Send a test message

```bash
bash setup.sh email send-test you@example.com
```

The command runs inside the api-gateway container and uses its configuration.
It sends **directly through the backend** and bypasses the outbox, rate limits
and the suppression list, so you get the result at once:

| Output | Meaning |
|--------|---------|
| `ok: accepted by the smtp backend (to y***@e***.com)` | The relay accepted the message. Now check the inbox and the headers (see above). |
| `transient failure: auth (to ...)` | The relay rejected the credentials. |
| `transient failure: tls (to ...)` | TLS handshake or certificate failure, or STARTTLS not offered. |
| `transient failure: connection` / `timeout` | Host or port unreachable, or blocked by a firewall. |
| `permanent failure: smtp_5xx` / `recipient_rejected` | The relay refused the message or the recipient. |
| `Email is disabled (EMAIL_BACKEND=none); nothing to test.` | The container still runs with email off. See [Apply the change](#apply-the-change). |
| `api-gateway container not running` | Start the stack first. |

Addresses are always shown masked. See
[Troubleshooting](#8-troubleshooting) for fixes.

### Outbox status

```bash
bash setup.sh email outbox status
```

```text
Outbox:
  pending    0
  sending    0
  sent       12
  dead       0
  expired    0
  cancelled  0
```

Add `--purpose password_reset` to filter by purpose, or pass a row id to see one
message's metadata (purpose, locale, status, attempts, last error class,
deadlines, masked recipient).

### The `/health` email object

```bash
curl -s http://localhost:8000/health | jq .email
```

A healthy SMTP setup:

```json
{
  "backend": "smtp",
  "status": "ok",
  "outbox_pending": 0,
  "oldest_pending_age_seconds": 0,
  "dead_last_24h": 0
}
```

| Field | Healthy | Notes |
|-------|---------|-------|
| `backend` | `smtp` | What the running process actually loaded. |
| `status` | `ok` | `disabled` when `EMAIL_BACKEND=none`. `degraded` when the backend check failed: the gateway could not connect, negotiate TLS, log in, or get a reply to `NOOP` from the relay. The check is cached for 60 s, and `/health` never waits more than about 2 s for it. |
| `outbox_pending` | small, near 0 | Messages waiting or in flight. |
| `oldest_pending_age_seconds` | well under 300 | A growing value means mail is not leaving. |
| `dead_last_24h` | `0` | Messages that gave up in the last 24 h. |

The object is **informational**: it never changes the gateway's own `/health`
status, so a relay outage does not mark the gateway unhealthy. The object never
contains the relay host, the username, the password, `EMAIL_FROM` or any
address. The counters are `null` only when the database could not be read.

### Alerts and metrics

The api-gateway exports Prometheus metrics, including
`minder_email_sent_total`, `minder_email_failed_total`, `minder_email_dead_total`,
`minder_email_expired_total`, `minder_email_skipped_total`,
`minder_email_outbox_pending`, `minder_email_outbox_oldest_pending_age_seconds`
and `minder_password_reset_requests_total{outcome}`.

An `email` rule group ships in Prometheus's `alerts.yml`. Every rule carries
`component="email"`, so you can route them with your existing Alertmanager
receivers:

| Alert | Fires when | Severity |
|-------|------------|----------|
| `EmailOutboxLag` | The oldest pending message has waited more than 5 min, for 10 min. | warning |
| `EmailOutboxLagSaaS` | The same condition in SaaS mode. | critical |
| `EmailDeadLetters` | Any message was dead-lettered in the last 15 min. | warning |
| `EmailSendFailureRatioHigh` | More than 20 % of send attempts failed over 15 min. | warning |

Make sure your Prometheus loads those rules: its `rule_files` must match where
`alerts.yml` is mounted in the container. Confirm that the group is loaded:

```bash
curl -s http://localhost:9090/api/v1/rules | jq -r '.data.groups[].name' | grep -x email
```

If that prints nothing, the email alerts are not active. See
[Monitoring](monitoring.md#alerting).

## 6. Turn on password reset

Self-service reset is **on as soon as email is on**: `PASSWORD_RESET_EMAIL_ENABLED`
defaults to `true`. To run email for notices only, set it to `false` and
[apply the change](#apply-the-change).

Check what the sign-in page offers:

```bash
curl -s http://localhost:8000/v1/auth/capabilities
# {"password_reset_email":true,"email_verification":false,"registration_mode":"invite"}
```

The client shows **Forgot password?** only when `password_reset_email` is
`true`. When it is `false`, the reset routes return `404`.

**The user's flow:**

1. The user selects **Forgot password?** on the sign-in page and enters their
   email address.
2. Whatever they entered, they see the same confirmation: "If an account exists
   for that address, we've sent instructions."
3. A local account receives a link to `/reset-password`. The link is valid for
   `PASSWORD_RESET_TOKEN_TTL_MINUTES` (15-60, default 30), counted from when the
   email is sent.
4. The user sets a new password (at least 8 characters). Every existing session
   for the account ends, and the user signs in again with the new password.
   They also get a "your password was changed" notice.

**What other accounts get:**

| Account | Result |
|---------|--------|
| SSO-linked | A notice that its password is managed by the identity provider, linking to `SSO_PASSWORD_HELP_URL` (or the IdP's issuer URL). **No reset link.** |
| Platform Admin, while reset by email is off for admins | A notice that points to the break-glass `bash setup.sh user reset-password` on the host. No reset link. |
| Deactivated | Nothing. |
| Address on the suppression list, a placeholder domain, or an invalid address | Nothing. |
| An address shared by several accounts (after normalization) | Nothing. List them with `bash setup.sh users email-collisions`. |
| No account | Nothing. |

Each of these outcomes is visible only to operators, through the audit log
(`auth.password_reset.*` entries) and the
`minder_password_reset_requests_total{outcome}` metric. The requester always
gets the same answer.

## 7. Operate

### The outbox and retries

Every email is first written to an outbox table **in the same database
transaction** as the change that triggered it. A rolled-back change sends
nothing, and a committed one is guaranteed an attempt. Each api-gateway replica
runs one worker. Workers claim rows with a lease, so replicas never send the same
row twice.

| Outcome | What happens |
|---------|--------------|
| Accepted by the relay | `sent` |
| Transient failure (SMTP 4xx, connection, timeout, TLS, **authentication**) | Retried with exponential backoff and jitter (about 30 s, 2 min, 8 min, 30 min, then at most 1 h between tries), up to 8 attempts. |
| Permanent failure (SMTP 5xx, invalid address, template error) | `dead` at once |
| Recipient rejected during the SMTP conversation (550/551/553 on `RCPT TO`) | `dead`, plus an automatic `hard_bounce` suppression. Policy rejections (`5.7.x`, for example "relay access denied") are treated as a plain 5xx and do not suppress the recipient. |
| Past its delivery deadline | `expired` |
| Recipient suppressed, or the reset link superseded | `cancelled` |

Authentication failures are retried, not dead-lettered. While you fix
credentials, queued mail waits instead of being thrown away, and
`EmailSendFailureRatioHigh` / `EmailOutboxLag` tell you something is wrong.

**Delivery deadlines.** A message is useful only for a limited time: **15 min**
for reset links and SSO/admin notices, and **24 h** for security notices. After
that, it expires instead of arriving late.

### Dead letters: retry and replay

```bash
bash setup.sh email outbox status                       # how many are dead?
bash setup.sh email outbox status <id>                  # why? (last_error_class)
bash setup.sh email outbox retry <id>                   # requeue one row
bash setup.sh email outbox replay --status dead         # requeue all dead rows
bash setup.sh email outbox replay --status dead --purpose password_changed --since 2026-10-01T00:00:00
```

Requeued rows get a fresh attempt budget. Output looks like:

```text
Requeued 3 row(s).
  skipped 2: past deliver before
```

Rows are **skipped** when they are not dead, are past their delivery deadline,
or have already had their address removed by retention. Because reset links
expire after 15 min, a dead reset email usually cannot be replayed. Fix the
cause, then ask the user to request a new link. Retries and replays are
audited.

### Suppressions

A suppressed address gets no mail: new messages to it are recorded as
`cancelled`/`suppressed`, and a reset request for it sends nothing. Addresses are
added:

- **automatically** (`hard_bounce`, source `smtp_sync`) when the relay rejects
  the recipient during sending;
- **manually** (`manual` or `complaint`, source `cli`), for example after a
  spam complaint.

Only rejections during the SMTP conversation are detected. Bounce messages
that your relay or provider sends back later are **not** processed, so handle
those in your provider's dashboard or add them manually.

```bash
bash setup.sh email suppressions list
#   3f9a0c41d2b7e865  hard_bounce smtp_sync 2026-10-02T09:14:03+00:00

bash setup.sh email suppressions add user@example.com                     # reason: manual
bash setup.sh email suppressions add user@example.com --reason complaint
bash setup.sh email suppressions remove user@example.com
# Unsuppressed u***@e***.com.   (or: "... is not suppressed; nothing changed.")
```

The list stores only a SHA-256 hash of the normalized address (trimmed, Unicode
NFKC, lower-cased), so `list` shows a hash prefix and never the address. To
check whether a specific address is in the list without changing anything:

```bash
printf '%s' 'user@example.com' | sha256sum | cut -c1-16    # compare with `list`
```

Remove a suppression only after the underlying problem is fixed, for example
after a mailbox that did not exist has been created. Changes are audited.

### Retention

Retention runs about hourly in the worker, **even when email is off**:

- `sent`, `expired` and `cancelled` rows have their address and template
  parameters cleared `EMAIL_OUTBOX_PII_RETENTION_DAYS` (7) days after they
  finish;
- `dead` rows keep a **masked** address (`u***@e***.com`) after that window, so
  you can still investigate them;
- every finished row is deleted `EMAIL_OUTBOX_RETENTION_DAYS` (30) days after it
  finishes;
- reset tokens are deleted `ACCOUNT_TOKEN_RETENTION_DAYS` (7) days after they
  expire or are used.

### Turning email off again

If the gateway starts with `EMAIL_BACKEND=none`, it **cancels every message
still queued** (`last_error_class=backend_disabled`) and invalidates their reset
links. If you turn email on again later, old links can never arrive late. The
gateway logs:

```text
Email is disabled: cancelled N queued email(s) (last_error_class=backend_disabled)
```

## 8. Troubleshooting

Start with `bash setup.sh email send-test <you>`. It reports the error class
directly. Logs carry row ids, purposes and error classes, never addresses or
links: `docker logs minder-api-gateway 2>&1 | grep -i email`.

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| Gateway won't start; log says `Refusing to start, email configuration: ...` | A missing or out-of-range setting, or a value refused in SaaS mode | Fix the named key in `.env` and [apply the change](#apply-the-change). |
| `transient failure: auth`; `EmailSendFailureRatioHigh` fires; outbox `pending` grows | Wrong `SMTP_USERNAME`/`SMTP_PASSWORD`, an API key used as the wrong field, a `$` in the password that was not escaped, or the provider needs the sender verified | Correct the credentials and apply the change. The queued mail is retried automatically. |
| `transient failure: tls` | Wrong mode for the port (`starttls` on 465, or `tls` on 587), a relay without STARTTLS, a self-signed or private-CA certificate, or `SMTP_HOST` not matching the certificate name | Use 587 with `starttls` or 465 with `tls`, set `SMTP_HOST` to the name on the certificate, and use a relay with a publicly trusted certificate. |
| `transient failure: connection` / `timeout` | Firewall, outbound port block (often port 25), wrong host | Test from the host (`nc -vz smtp.provider.example 587`) and switch to 587/465. |
| `permanent failure: smtp_5xx` | The relay refuses the sender (domain not verified, relay access denied) | Verify the `EMAIL_FROM` domain with the provider, or allow this host on your relay. |
| `send-test` says `ok`, but nothing arrives | Spam folder, provider-side hold, or a rejection after the relay accepted the message | Check spam and the provider's activity log. Check the [sender DNS](#4-sender-dns-spf-dkim-dmarc) records. |
| Mail lands in spam | SPF/DKIM/DMARC missing or not aligned with the `EMAIL_FROM` domain, or a new domain/IP with no reputation | Publish all three records, confirm `dmarc=pass` in the headers, and use a domain you control in `EMAIL_FROM`. |
| `/health` shows `"backend": "none"` after you configured SMTP | The container still runs with the old environment | Recreate it: `bash setup.sh start` (not `restart api-gateway`). |
| Messages `cancelled` with `backend_disabled` | The gateway started once with email off (for example, before `.env` was applied) | Expected. Fix the configuration and ask users to request a new reset link. |
| `EmailOutboxLag` fires, `status` is `ok` | Sends are throttled by `EMAIL_MAX_PER_MINUTE`, or retries are backing off | Check `email outbox status`. Raise `EMAIL_MAX_PER_MINUTE` if your provider allows it. |
| Reset requests return `429` | The per-IP limit on the reset routes. Many users behind one NAT can hit it. | Raise `AUTH_EMAIL_IP_LIMIT_PER_MINUTE`, or set it to `0` to rely on the per-account and per-recipient limits. |
| A user didn't get a reset email | See the checklist below | |

**A user didn't get a reset email:**

1. `curl -s http://localhost:8000/v1/auth/capabilities`: is `password_reset_email`
   `true`?
2. Check the audit log for that account's latest `auth.password_reset.*` entry:
    - `requested`: a link was queued. Continue with step 3.
    - `sso_notice` / `privileged_refused`: the user got a notice instead of a
      link.
    - `undeliverable`: a suppressed, placeholder or invalid address.
    - `rejected` with reason `account_inactive`: the account is deactivated.
    - `suppressed` with reason `account_limit` or `recipient_cap`: the user
      asked too often. The limits are 3 per hour and one per 60 s per account,
      and 10 per address per day.
    - No entry at all: no account matched that address, or the address is
      shared by several accounts (`bash setup.sh users email-collisions`).
3. `bash setup.sh email outbox status --purpose password_reset`: look for
   `dead` (inspect with `outbox status <id>`) or `expired`. An expired reset
   email was not delivered within 15 min, usually because of a relay outage.
4. Check the user's spam folder.
5. In every case, the user can simply request a new link once the cause is
   fixed. An administrator can also reset the password directly (see
   [Authentication](authentication.md#password-management)).

## 9. Security notes

- **No account enumeration.** The reset request always returns `202` with the
  same body, whether or not the address has an account, is SSO-linked,
  deactivated, suppressed or rate-limited. The response is padded to
  `PASSWORD_RESET_MIN_RESPONSE_MS` plus random jitter, so the branch that queues
  mail is not measurably slower. Only malformed input (`422`) and the per-IP
  limit (`429`) differ, and neither depends on the account. Rate limits skip
  silently for the same reason.
- **Short-lived, single-use tokens.** The reset secret is created right before
  the email is sent. It is valid for `PASSWORD_RESET_TOKEN_TTL_MINUTES`
  (15-60 min) and works only once. It is also bound to the account's current
  password: any password change invalidates outstanding links, and a successful
  reset voids the account's other reset links. Only a hash of the token is
  stored.
- **Link handling.** The token travels in the URL **fragment**
  (`#token=...`), so it never reaches server access logs or a `Referer`
  header. The client submits it with an explicit POST. Every token failure
  returns the same `400 invalid_or_expired_token`.
- **A reset ends all sessions.** A completed reset revokes every existing
  session for the account, on every device, and does **not** sign the user in.
  The account owner gets a "password changed" notice, so an unexpected reset
  is noticed.
- **SSO and privileged accounts get no links.** SSO-linked accounts are pointed
  to their identity provider. With Platform Admin 2FA required (always in SaaS),
  Platform Admins get a notice instead of a link, and their recovery is the
  host-side break-glass command.
- **No secrets or addresses in output.** Logs, CLI output, audit metadata and
  `/health` show masked addresses only. The `log` backend is the one deliberate
  exception, which is why it is for development only. Keep `SMTP_PASSWORD` in
  `.env` (mode `600`), as with every other secret; see [Security](security.md).

## Related pages

- [Production](production.md): go-live checklist and operations.
- [Authentication](authentication.md): accounts, admin password resets,
  sessions.
- [Security](security.md): secrets handling and hardening.
- [Monitoring](monitoring.md): Prometheus, alert rules, Alertmanager receivers.
