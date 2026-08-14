# Configuring your box

Everything an operator sets on a running workinabox box, grouped by what you are trying to
achieve.

Settings are environment variables read once when the backend starts, so **a change takes effect
on restart**. On a deployed box they live in `/etc/wiab/wiab.env`.

Booleans accept `1`, `true`, `yes` or `on`. Anything else — including unset, empty, or `false` —
is off.

> This page covers the settings a box is meant to be tuned with. The complete list of every
> variable the system reads, including internal ones, is a developer reference:
> [`../Development/ENVIRONMENT_VARIABLES.md`](../Development/ENVIRONMENT_VARIABLES.md).

## Public address and TLS

| Setting | Default | Purpose |
| --- | --- | --- |
| `WIAB_BASE_URL` | `http://localhost:3000` | The origin your users type in. |
| `WIAB_TLS_CERT` / `WIAB_TLS_KEY` | unset | Paths to your certificate and key, in PEM. |

**`WIAB_BASE_URL` is the most important setting on the box.** It decides whether session cookies
are marked `Secure`, and it is the base for every link in outgoing email and every OAuth redirect
URI. If it doesn't match the address users actually visit, logins and email links break in ways
that look unrelated to this setting.

With both TLS paths unset the backend generates a self-signed certificate. That is fine for a
trial and wrong for anything real — browsers will warn, and some clients will refuse outright.

## Who can sign in

| Setting | Default | Purpose |
| --- | --- | --- |
| `WIAB_AUTH_LOCAL_SIGNUP` | off | Let anyone create their own account. |
| `WIAB_AUTH_GOOGLE_ENABLED` | off | Offer "Continue with Google". |
| `WIAB_GOOGLE_CLIENT_ID` / `WIAB_GOOGLE_CLIENT_SECRET` | empty | Google OAuth client. |
| `WIAB_AUTH_OIDC_ENABLED` | off | Offer enterprise SSO. |
| `WIAB_OIDC_ISSUER` / `WIAB_OIDC_CLIENT_ID` / `WIAB_OIDC_CLIENT_SECRET` | empty | Your identity provider. |

With everything off, the box runs invite-only: the owner signs in and invites colleagues, who set
a password through an emailed link. That is the default and it is a reasonable place to stay.

Setting up the third-party side:

- [`../Installation/GOOGLE_OAUTH.md`](../Installation/GOOGLE_OAUTH.md)
- [`../Installation/MICROSOFT_ENTRA.md`](../Installation/MICROSOFT_ENTRA.md) — or any other OIDC
  provider

Turning a flag on without its credentials shows the button and fails the login, so set both
together.

## Email

Invitations, password resets and address verification all need email. **Until you configure a
provider, these messages are written to the server log instead of being sent** — the box appears
to work and no one receives anything.

| Setting | Default | Purpose |
| --- | --- | --- |
| `WIAB_EMAIL_PROVIDER` | `resend` | `resend` or `smtp`. |
| `WIAB_EMAIL_FROM` | `no-reply@workinabox.local` | From address. |
| `RESEND_API_KEY` | unset | Required for `resend`. |
| `WIAB_SMTP_HOST` | unset | Required for `smtp`. |
| `WIAB_SMTP_PORT` | `587` | |
| `WIAB_SMTP_USER` / `WIAB_SMTP_PASSWORD` | unset | Optional. |
| `WIAB_SMTP_TLS` | off | |

Full setup, including how to tell from the startup log which transport was chosen:
[`../Installation/EMAIL_DELIVERY.md`](../Installation/EMAIL_DELIVERY.md).

## The first administrator

| Setting | Default | Purpose |
| --- | --- | --- |
| `WIAB_DEV_OWNER_PASSWORD` | none | Password for the seeded Owner account. |
| `WIAB_BOOTSTRAP_TOKEN_FILE` | unset | Path to capture the one-time bootstrap token. |

On first boot against an empty database the box seeds an organization and an Owner.

**There is no default password.** A box whose `WIAB_BASE_URL` is not local refuses to start
without `WIAB_DEV_OWNER_PASSWORD` — a built-in default would be a published credential on every
deployment that forgot to change it.

The bootstrap token is short-lived and, by default, only written to the log. Point
`WIAB_BOOTSTRAP_TOKEN_FILE` at a path to capture it in a file instead (created `0600`).

## Storage and git hosting

| Setting | Default | Purpose |
| --- | --- | --- |
| `WIAB_GIT_ROOT` | a temporary directory | Where hosted git repositories live. |
| `WIAB_GIT_SSH_HOST_KEY` | unset | Path to a persistent SSH host key. |
| `WIAB_GIT_SSH_ADDR` | `0.0.0.0:2222` | Where the git SSH transport listens. |
| `WIAB_DATA_DIR` | `$HOME/.local/share/wiab` | Data directory, including model files. |
| `DATABASE_URL` | local Postgres | Database connection. |

**Set `WIAB_GIT_ROOT`.** The default is a temporary directory, and hosted repositories will not
survive a reboot.

**Set `WIAB_GIT_SSH_HOST_KEY` too** — in fact a box with a non-local `WIAB_BASE_URL` refuses to
start without it. A regenerated host key trips every client's known-hosts check and looks, to
your users, exactly like a machine-in-the-middle warning.

## Meetings

| Setting | Default | Purpose |
| --- | --- | --- |
| `WIAB_MEDIASOUP_ANNOUNCED_ADDRESS` | `10.0.2.2` | The address clients are told to send media to. |
| `WIAB_MEDIASOUP_MIN_PORT` / `WIAB_MEDIASOUP_MAX_PORT` | `40000` / `40999` | UDP range for media. |
| `WIAB_MEDIASOUP_LISTEN_IP` | `0.0.0.0` | Interface to bind. |

The announced address must be the address clients can actually reach — your public IP when the
box is behind NAT, otherwise its LAN address. Get this wrong and calls connect, then carry no
audio.

**Your firewall must open exactly the UDP port range configured here.** Two transports per
participant, so the default range carries roughly 500 concurrent participants.

### Transcription and summaries

| Setting | Default | Purpose |
| --- | --- | --- |
| `WIAB_WHISPER_ENABLED` | off | Speech-to-text for meetings. |
| `WIAB_WHISPER_MODEL_FILE` | — | Required when enabled. |
| `WIAB_LLAMA_ENABLED` | off | Meeting summaries and minutes. |
| `WIAB_LLAMA_MODEL_FILE` | — | Required when enabled. |
| `WIAB_STT_LANGUAGE` | auto-detect | e.g. `en`. |

Both run locally — audio and transcripts do not leave the box. Model files are read from
`<data dir>/models/`. **Enabling a model whose file is missing stops the backend from starting**,
which is deliberate: a silently disabled transcriber is worse than a box that won't boot.

Tuning knobs for context size, reply length and thread counts exist but rarely need changing;
they are in the developer reference.

## Logging

| Setting | Default | Purpose |
| --- | --- | --- |
| `RUST_LOG` | `wiab=info,tower_http=info` | Log verbosity. |

Use `wiab=debug` when diagnosing a problem. The startup log is worth reading after any change
here — the backend states which email transport, authentication connections and models it
actually ended up with, which is the fastest way to catch a setting that didn't apply.
