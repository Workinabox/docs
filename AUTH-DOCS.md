# Authentication & Identity (authbox)

This document describes the identity, authentication, and access system built for
Workinabox — what exists, how it works, how to configure it for different deployments, and
what is left to do. It is the source of truth for the auth system; the older
[IDENTITY_AND_ACCESS_PLAN.md](IDENTITY_AND_ACCESS_PLAN.md) describes the original git-auth
RBAC work that this builds on.

The system is packaged as a **reusable, product-neutral crate set** (`authbox`) so the same
capability can drop into a second product (TruthDB) — see [Reuse](#reusing-authbox-in-another-product).

---

## Table of contents

1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Data model & migrations](#data-model--migrations)
4. [How each flow works](#how-each-flow-works)
5. [HTTP API reference](#http-api-reference)
6. [Configuration (environment variables)](#configuration-environment-variables)
7. [Usage scenarios](#usage-scenarios)
8. [Security notes](#security-notes)
9. [Reusing authbox in another product](#reusing-authbox-in-another-product)
10. [Testing & verification](#testing--verification)
11. [What is not done yet](#what-is-not-done-yet)

---

## Overview

Workinabox supports three ways for a **human** to authenticate, all resolving to one local
user record, plus the pre-existing **machine** credentials:

| Method | What it is |
|---|---|
| **Local email + password** | argon2id-hashed password, server-side browser sessions. |
| **Social login** | "Continue with Google" (OIDC). MS/Apple slot into the same path later. |
| **Enterprise SSO** | Inbound OIDC federation to a customer's own IdP (Okta, Entra, …). |
| **Machine (unchanged)** | Personal access tokens (PATs) over HTTPS Basic/Bearer, and SSH keys — used by agents, CI, and `git`. |

Around those, it provides: change-password, forgotten-password reset (emailed link), admin
**invite**, self-service **signup with email verification**, user **deactivate/activate**,
and a generic **role-based access** core.

Everything that is security-critical (password hashing, OIDC token validation) is done with
vetted libraries (`argon2`, `openidconnect`), not hand-rolled. All identity data lives in
the product's own Postgres — there is no external identity provider.

---

## Architecture

### The crate set

The reusable identity lives in three crates that depend only on each other (never on
Workinabox), mirroring the host's `core / app / inf` layering:

```
authbox-core   domain + ports (no I/O): Role/Operation/RBAC, PrincipalId, Session,
               PasswordCredential, FederatedIdentity, VerificationToken, and the *traits*
               (SessionStore, CredentialStore, PasswordHasher, OidcPort, EmailSender,
               UserDirectory, Clock, …).
authbox-app    application services: AuthenticationService, FederationService,
               PasswordResetService, InvitationService.
authbox-inf    adapters: Argon2idPasswordHasher, RandomSecretGenerator, OidcRelyingParty
               (openidconnect), Logging/Smtp EmailSender, in-memory + Postgres stores,
               and the SQL migrations.
```

### Auth via ports (the key design decision)

`authbox` does **not** own Workinabox's `User`. Instead it owns the *authentication
machinery*, keyed on an opaque **`PrincipalId`** (just the user-id string, e.g. `"U-1"`),
and reaches the host's user store through one trait:

```rust
trait UserDirectory {
    async fn find_by_email(&self, email: &str) -> Result<Option<PrincipalId>, AuthError>;
    async fn provision(&self, email: &str, name: &str) -> Result<PrincipalId, AuthError>;
}
```

Workinabox implements this (`wiab-inf/src/wiab_user_directory.rs`) over its existing
`UserApplicationService`. `find_by_email` returns **only active** users, so lifecycle policy
(pending/deactivated cannot log in) stays host-side. This is why TruthDB can reuse the whole
thing without adopting Workinabox's `User` type — it just implements `UserDirectory` over its
own users.

The generic **RBAC** was extracted the same way: `Role`/`Operation` and an `effective_role`
policy over opaque `ResourceRef { kind, id }` live in `authbox-core`; Workinabox keeps its
`Org ⊇ Project ⊇ Repo` containment as a `ResourceHierarchy` strategy
(`wiab-core/src/access/access_policy.rs`).

### Where it plugs into Workinabox

- **`bootstrap.rs`** constructs every authbox service from env config and the chosen
  persistence backend, and puts them in `AppState`.
- **`http_api.rs`** mounts the `/auth/*` routes and — importantly — `authenticate()` gained
  a **session-cookie fallback** *after* the existing bearer/Basic token path. Precedence is
  **bearer token first** (machine clients, `git`), then session cookie (browsers). Git-over-
  HTTP, git-over-SSH, and PATs are unchanged.
- **`SystemClock`** implements both the meeting clock and the authbox `Clock`.

---

## Data model & migrations

authbox owns its tables and runs its own migrations in a **separate refinery history table**
(`authbox_migrations`), so its schema is independent of the host's and extracts cleanly.
`authbox_inf::run_migrations(&pool)` is called alongside the host's in `bootstrap.rs`.

**authbox tables** (`authbox-inf/migrations/`):

| Table | Holds |
|---|---|
| `user_password` | `(user_id PK, phc_hash, state, updated_at)` — one argon2id PHC string per principal. |
| `auth_session` | Server-side sessions: `token_hash` (hash of the cookie secret), `csrf_hash`, `principal_id`, created/last-seen/idle-expiry/absolute-expiry, `revoked`. |
| `federated_identity` | `(issuer, subject) → principal_id` links (Google / enterprise). |
| `auth_flow` | Short-lived OIDC login state (PKCE verifier, nonce, return_to), single-use by `state`. |
| `verification_token` | Single-use, expiring tokens for reset / invite / email-verify (`token_hash PK`, `purpose`, `principal_id`, `expires_at`). |

**Workinabox user tables** (`wiab-inf/migrations/`, added by this work):

- `app_user.state` (`V11`) — `'active' | 'pending' | 'deactivated'`.
- `user_external_ref` (`V10`) — replaced the old `app_user.agent_id` column; generic
  `(system, ref_id)` links (e.g. `("agent", "A-9")`), SCIM-ready.

Secrets are **never stored in the clear**: passwords are argon2id PHC strings; session,
CSRF, reset, invite and verification secrets are stored only as SHA-256 hashes — the
plaintext travels once (in a cookie or an emailed link).

---

## How each flow works

### Local login & sessions
1. `POST /auth/session {email, password}` → `AuthenticationService::login_with_password`:
   resolve email → principal (active only), load the password credential, argon2id-verify
   (off the async worker via `spawn_blocking`).
2. On success, establish a session: mint a random cookie secret + CSRF secret, store their
   hashes + idle (8 h) and absolute (7 d) expiries, set the cookie, return `{user, csrf_token}`.
3. The cookie (`wiab_session`) is `HttpOnly`, `SameSite=Lax`, `Path=/`, and `Secure` when
   `WIAB_BASE_URL` is `https`. Every subsequent request resolves it in `authenticate()`; the
   idle window slides on use; sessions are revocable (logout, password reset, deactivate).

### Change password (logged in)
`PUT /auth/password {current_password, new_password}` re-verifies the current password, then
sets the new one. Existing sessions are kept (voluntary change, not a compromise reset).

### Forgotten-password reset (emailed link)
1. `POST /auth/password/reset/request {email}` always returns **202** (no account-existence
   disclosure). If the email is a known active user, a single-use token is stored (hashed,
   1 h expiry) and a link `…/reset-password?token=…` is emailed.
2. `POST /auth/password/reset/confirm {token, new_password}` consumes the token, sets the new
   password, and **revokes all the user's sessions** (a reset implies possible compromise).

### Social login & enterprise SSO (OIDC federation)
Google and an enterprise IdP are the **same code path** with different config.
1. `GET /auth/oidc/{connection}/start?next=…` → `FederationService::begin_login` →
   `OidcRelyingParty::begin`: discover the issuer, build the authorization URL with a fresh
   **PKCE** challenge + **nonce**, persist them in `auth_flow` keyed by `state`, and 302 the
   browser to the IdP.
2. `GET /auth/oidc/{connection}/callback?code&state` → consume the `auth_flow` row (single
   use = the CSRF/state check), exchange the code (with the stored PKCE verifier), and
   **validate the ID token** (signature via the issuer's JWKS, `iss`/`aud`/`exp`, and the
   stored nonce) — all done by `openidconnect`.
3. Resolve the identity: existing `(issuer, subject)` link → log in; else (with a verified
   email) either link to an existing user (**enterprise only**, `auto_link_verified_email`)
   or just-in-time provision a new one (`UserDirectory::provision`). Then establish a session
   and 302 to `next` (validated to be a same-origin relative path — no open redirect).

`subject` (the IdP's stable id) is the durable key, never the email. Google is configured
with `auto_link_verified_email = false` to prevent account takeover via an attacker-asserted
email; the enterprise connection sets it `true` because its users are pre-provisioned.

### Admin invite
1. Owner calls `POST /users/invite {email, name}` → create a **pending** user (no password) →
   `InvitationService::invite` issues an `Invite` token and emails `…/accept-invite?token=…`.
2. The invitee `POST /auth/invite/accept {token, password}` → set their first password →
   **activate** the user → auto-login (session established).

### Self-service signup + email verification (off by default)
Only when `WIAB_AUTH_LOCAL_SIGNUP=true`:
1. `POST /auth/signup {email, password, name}` → create a **pending** user, set the password,
   and email `…/verify-email?token=…`. Always returns **202** (no existence leak).
2. `POST /auth/verify-email {token}` → consume the token → **activate** → auto-login.

Because pending users are not active, they cannot log in until they click the link.

### Deactivate / reactivate
`POST /users/{id}/deactivate` (owner) sets the user `deactivated` and **revokes their
sessions immediately**; `find_by_email` then excludes them so they can't log in (their role
grants and history are preserved). `POST /users/{id}/activate` reverses it.

### Machine auth (unchanged)
PATs (`wiab_pat_…`, SHA-256-hashed, scoped, expirable) over HTTPS Basic/Bearer and SSH public
keys continue to work exactly as before. Browsers use sessions; machines use tokens/keys; the
bearer path takes precedence so the two never collide.

---

## HTTP API reference

All paths are under the backend root and reached by the SPA as `/api/...` (nginx strips the
`/api` prefix). "Owner" = caller holds the Owner role; "authed" = any valid session or token.

| Method & path | Auth | Purpose |
|---|---|---|
| `POST /auth/session` | public | Local login → sets cookie, returns `{user, csrf_token}`. |
| `GET /auth/session` | authed | Current user (`{id, name, email, is_owner}`). |
| `DELETE /auth/session` | session | Logout (revokes + clears cookie). |
| `PUT /auth/password` | authed | Change own password. |
| `POST /auth/password/reset/request` | public | Email a reset link (always 202). |
| `POST /auth/password/reset/confirm` | public | Set a new password from a reset token. |
| `GET /auth/config` | public | `{local_password, signup, google, oidc}` — which buttons to show. |
| `GET /auth/oidc/{conn}/start` | public | Begin OIDC login (302 to IdP). |
| `GET /auth/oidc/{conn}/callback` | public | Finish OIDC login (302 to `next`). |
| `POST /auth/signup` | public¹ | Self-service signup (pending + verify email). |
| `POST /auth/invite/accept` | public | Accept an invite (set password, auto-login). |
| `POST /auth/verify-email` | public | Confirm signup email (auto-login). |
| `POST /users/invite` | owner | Create a pending user + email an invite. |
| `POST /users/{id}/deactivate` | owner | Disable login + revoke sessions. |
| `POST /users/{id}/activate` | owner | Re-enable a deactivated user. |
| `GET/POST /users`, `GET /users/{id}` | owner / self-or-owner | List/create/get users (create = active). |
| `…/users/{id}/ssh-keys`, `…/users/{id}/tokens` | self-or-owner | Manage SSH keys + PATs (machine creds). |
| `GET/POST/DELETE /role-assignments` | owner | Grant/list/revoke roles. |

¹ Returns 403 unless `WIAB_AUTH_LOCAL_SIGNUP=true`.

The SPA pages are: `/login`, `/signup`, `/forgot-password`, `/reset-password`,
`/accept-invite`, `/verify-email`, and the in-console `/account` (change password) and
`/users` (admin: create/invite/deactivate, plus SSH keys & PATs).

---

## Configuration (environment variables)

| Variable | Default | Meaning |
|---|---|---|
| `WIAB_PERSISTENCE` | `postgres` | `postgres` or `memory` (memory loses sessions/users on restart). |
| `DATABASE_URL` | `postgres://wiab:wiab@localhost:5432/wiab` | Postgres connection. |
| `WIAB_BASE_URL` | `http://localhost:3000` | Public origin. Drives cookie `Secure` (https ⇒ Secure) and all email / OIDC-redirect URLs. **Set this in production.** |
| `WIAB_DEV_OWNER_PASSWORD` | `owner` | Password seeded for the bootstrap owner (`owner@workinabox.local`). |
| `WIAB_AUTH_LOCAL_SIGNUP` | `false` | Enable self-service signup. |
| `WIAB_AUTH_GOOGLE_ENABLED` | `false` | Offer "Continue with Google". |
| `WIAB_GOOGLE_CLIENT_ID` / `WIAB_GOOGLE_CLIENT_SECRET` | — | Google OAuth client. Redirect URI: `${WIAB_BASE_URL}/api/auth/oidc/google/callback`. |
| `WIAB_AUTH_OIDC_ENABLED` | `false` | Offer enterprise SSO. |
| `WIAB_OIDC_ISSUER` / `WIAB_OIDC_CLIENT_ID` / `WIAB_OIDC_CLIENT_SECRET` | — | The customer's IdP. Redirect URI: `${WIAB_BASE_URL}/api/auth/oidc/enterprise/callback`. |
| `WIAB_SMTP_HOST` | — | If set, email is delivered via SMTP; otherwise it is **logged** (the link appears in the server log). |
| `WIAB_SMTP_PORT` | `587` | SMTP port. |
| `WIAB_SMTP_USER` / `WIAB_SMTP_PASSWORD` | — | SMTP credentials (optional). |
| `WIAB_SMTP_FROM` | `no-reply@workinabox.local` | From address. |
| `WIAB_SMTP_TLS` | `false` | Use TLS (rustls). Leave off for a local Mailpit on `:1025`. |

Booleans accept `1/true/yes/on`. Flags off ⇒ the corresponding login buttons are hidden and
the endpoints reject.

> **Secrets:** these land on disk like `db_password` does today (env file / Terraform). A
> dedicated secrets manager is future work; at minimum generate the values out-of-band and
> keep the env file `0600`. (There is no separate session-signing key — sessions are opaque
> server-side records, so there is nothing to forge if the DB is intact.)

---

## Usage scenarios

### A. Single company, self-hosted box (the default)
Nothing to configure beyond `WIAB_BASE_URL`. The bootstrap owner can log in
(`owner@workinabox.local` / `WIAB_DEV_OWNER_PASSWORD`), then **invite** colleagues from the
Users page; they set a password via the emailed link. Signup and federation stay off. Set up
SMTP (below) so invites/resets actually send.

### B. Real email
Point at any SMTP server: `WIAB_SMTP_HOST=smtp.example.com`, `WIAB_SMTP_PORT=587`,
`WIAB_SMTP_USER`/`WIAB_SMTP_PASSWORD`, `WIAB_SMTP_FROM=you@example.com`, `WIAB_SMTP_TLS=true`.
Without it, everything still works but the links are written to the log instead of emailed
(fine for dev; the local docker-compose wires a Mailpit inbox at `http://localhost:8025`).

### C. Google login
Create an OAuth client in Google Cloud, set the redirect URI to
`${WIAB_BASE_URL}/api/auth/oidc/google/callback`, then set `WIAB_AUTH_GOOGLE_ENABLED=true`,
`WIAB_GOOGLE_CLIENT_ID`, `WIAB_GOOGLE_CLIENT_SECRET`. A "Continue with Google" button appears.
A Google user with no local account is provisioned just-in-time; a Google email that matches
an existing **password** account is *not* auto-linked (the user must sign in locally and link
— a deliberate anti-takeover choice).

### D. Enterprise SSO (customer's IdP)
The customer registers Workinabox as an OIDC client in their IdP (Okta/Entra/…), with
redirect URI `${WIAB_BASE_URL}/api/auth/oidc/enterprise/callback`. Set
`WIAB_AUTH_OIDC_ENABLED=true`, `WIAB_OIDC_ISSUER` (their issuer URL),
`WIAB_OIDC_CLIENT_ID`, `WIAB_OIDC_CLIENT_SECRET`. Their users click "Sign in with SSO";
because this connection auto-links verified emails, a user the admin pre-created (or invited)
is matched, and otherwise provisioned just-in-time. One IdP connection per deployment (one
box per company).

### E. Open self-service signup
Set `WIAB_AUTH_LOCAL_SIGNUP=true` (and SMTP). A "Create an account" link appears on login;
new users are created **pending** and must click the verification email to activate. (Domain
restriction is not yet implemented — see [What is not done](#what-is-not-done-yet).)

---

## Security notes

- **Password hashing:** argon2id at the OWASP baseline (m = 19 MiB, t = 2, p = 1), run off
  the async worker. Verify is constant-time (the `argon2` crate).
- **Sessions:** opaque server-side records (instant revocation), `HttpOnly` + `SameSite=Lax`
  + `Secure` (in https) cookies. Created only after auth (no fixation). A CSRF token is
  issued for future double-submit hardening; today `SameSite=Lax` is the CSRF defense
  (the console is same-origin behind the `/api` proxy).
- **OIDC:** authorization-code **+ PKCE**, server-side single-use `state`, nonce-checked ID
  tokens, JWKS signature validation — all via `openidconnect`. `return_to` is restricted to
  same-origin relative paths (no open redirect). HTTPS to the IdP uses rustls (no OpenSSL).
- **Anti-enumeration:** password-reset request and signup always return the same response
  regardless of whether the email exists.
- **Tokens/secrets** are stored hashed; reset/invite/verify links are single-use and expire
  (reset 1 h, invite/verify 24 h).
- **Machine vs. human:** bearer/Basic token wins over the cookie, so CSRF (a cookie concern)
  never applies to API/`git` callers, and sessions never authorize a `git push`.

---

## Reusing authbox in another product

The `authbox-*` crates depend on nothing from Workinabox. To use them elsewhere (e.g.
TruthDB):

1. Depend on `authbox-core/app/inf`.
2. Implement **`UserDirectory`** over your own user store (`find_by_email` should return only
   users allowed to log in; `provision` JIT-creates one).
3. For RBAC, implement **`ResourceHierarchy`** for your resource model and map your scopes to
   `ResourceRef`.
4. Construct the services (`AuthenticationService`, `FederationService`,
   `PasswordResetService`, `InvitationService`) with the `authbox-inf` adapters (or your own),
   run `authbox_inf::run_migrations`, and mount HTTP handlers like Workinabox's
   `http_api.rs`. (The HTTP handlers themselves currently live in the host; if a second
   consumer wants them shared, factor them into an `authbox-inf` Axum router then.)

The crates are currently developed inside the Workinabox Cargo workspace; extracting them to
their own repo is a path-dep → version-dep rename plus moving the three directories.

---

## Testing & verification

- **Unit tests** (run with `cargo test`): the RBAC policy, session/credential domain,
  `AuthenticationService` (login→resolve→logout, wrong/unknown credentials),
  `FederationService` (JIT-provision, enterprise link, Google no-silent-link, unverified
  email, bad state), `PasswordResetService` (request→email→confirm, anti-enumeration),
  `InvitationService` (invite→accept, signup-verify, wrong purpose), and the argon2/secret
  adapters. ~330 tests, all green.
- **Docker integration tests** (real services, `--ignored`). These run **in CI** — the
  `test` job (`.github/workflows/ci.yml`) provisions `postgres` + `mailpit` + `oidc-mock`
  service containers and runs all three (host/port/issuer are passed via env). To run them
  locally:
  - **Postgres migrations + persistence:** `docker run -d --rm -p 5432:5432 -e POSTGRES_USER=wiab -e POSTGRES_PASSWORD=wiab -e POSTGRES_DB=wiab postgres:16`
    then `DATABASE_URL=postgres://wiab:wiab@localhost:5432/wiab cargo test -p wiab-inf --test postgres_integration -- --ignored`
    — applies both migration series (host V1–V11 + authbox V1–V3) idempotently on a fresh DB
    and exercises the repos.
  - **Email:** `docker run -d --rm -p 1025:1025 -p 8025:8025 axllent/mailpit` then
    `cargo test -p authbox-inf --test smtp_mailpit -- --ignored` — proves the SMTP sender
    actually delivers.
  - **OIDC:** `docker run -d --rm -p 9090:8080 ghcr.io/navikt/mock-oauth2-server:2.1.10` then
    `cargo test -p authbox-inf --test oidc_mock -- --ignored` — a full auth-code + PKCE
    round-trip against a real OIDC server, including JWKS/nonce validation and rejecting a
    tampered nonce.
- **Local full stack:** `docker compose -f dev/local/docker-compose.yml up --build` brings up
  Postgres, the backend, the frontend, Mailpit (`:8025`), and a mock OIDC server (`:9090`),
  with SMTP pre-wired so reset/invite emails are viewable immediately.

---

## What is not done yet

- **OIDC adapter is not exercised end-to-end in production** — it is verified against the
  mock IdP and compiles against the real `openidconnect` API, but a first real Google /
  enterprise login should be smoke-tested with real credentials. (In docker-compose, a mock
  IdP has a browser-vs-backend hostname subtlety, noted inline there; real IdPs don't.)
- **Microsoft / Apple** social login — not wired (they go through the same OIDC connection
  mechanism; Apple needs its key-based client-secret JWT).
- **SCIM** automated provisioning — the user record is shaped for it (`external_refs`,
  lifecycle state) but no SCIM endpoint exists; JIT + invite + admin-create cover provisioning.
- **SAML** — OIDC only.
- **MFA / TOTP** — not implemented.
- **Signup domain allow-list** — signup is gated by a flag but not restricted to specific
  email domains yet.
- **CSRF double-submit enforcement** — the token is issued but not yet required (relying on
  `SameSite=Lax`); enforce it for defense-in-depth.
- **Login-timing anti-enumeration** — `login` skips the argon2 verify for an unknown email
  (a minor timing oracle); flatten it with a dummy hash.
- **Secrets management** — env-file/Terraform posture; no vault.
- **Idle-session "touch" throttling** — the idle window currently updates on every request;
  throttle the write under load.
