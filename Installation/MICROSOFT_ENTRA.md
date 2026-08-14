# Installing enterprise SSO with Microsoft Entra

How to register workinabox as an application in Microsoft Entra ID and point a backend at it.
Entra is the common enterprise case, but nothing here is Entra-specific beyond the issuer URL
shape — any OIDC provider works through the same `enterprise` connection.

Related: [`../Configuration/SETTINGS.md`](../Configuration/SETTINGS.md) for the settings this
runbook produces, alongside everything else a box is configured with.

## Before you start

You need permission to create app registrations in the target Entra tenant, and you need to
know the public base URL of the box (`WIAB_BASE_URL`) — the redirect URI is derived from it and
must match byte-for-byte.

One registration per box. Each company runs its own backend against its own tenant.

## Register the application

1. Sign in to the [Microsoft Entra admin center](https://entra.microsoft.com). If you belong to
   several tenants, use the top-bar **Settings** (gear) to switch to the target tenant.
2. Go to **Entra ID → App registrations → New registration**.
3. **Name:** `Workinabox`.
   **Supported account types:** *Single tenant only — `<your tenant>`*.
4. **Redirect URI:** platform **Web**, value:

   ```text
   ${WIAB_BASE_URL}/api/auth/oidc/enterprise/callback
   ```

   For the local dev stack that is `http://localhost:3000/api/auth/oidc/enterprise/callback`.
   Plain `http` is accepted **only** for `localhost`; every deployed environment must be HTTPS.

   The path is not configurable — the backend builds it as
   `<WIAB_BASE_URL>/api/auth/oidc/enterprise/callback`, where `enterprise` is the fixed slug of
   the enterprise federation connection.
5. Click **Register**.
6. On **Overview**, copy the **Application (client) ID** and the **Directory (tenant) ID**.
7. Go to **Manage → Certificates & secrets → Client secrets → New client secret**. Copy the
   **`Value`** immediately — it is shown once, and the `Secret ID` is *not* the secret. Treat it
   like a password.

   Client secrets **expire**. Set a reminder to rotate before the expiry you chose.

## Configure the backend

Map the values you copied onto four environment variables:

| Entra value | Environment variable |
| --- | --- |
| `https://login.microsoftonline.com/<Directory (tenant) ID>/v2.0` | `WIAB_OIDC_ISSUER` |
| Application (client) ID | `WIAB_OIDC_CLIENT_ID` |
| Client secret `Value` | `WIAB_OIDC_CLIENT_SECRET` |
| — | `WIAB_AUTH_OIDC_ENABLED=true` |

All four are required. The connection is registered at startup only when
`WIAB_AUTH_OIDC_ENABLED` is set to a truthy value (`1`/`true`/`yes`/`on`); the other three
default to empty, so a half-configured box silently runs with enterprise SSO off.

The backend requests the `openid`, `email` and `profile` scopes.

**Where to put them.** Locally, in `dev/local/oidc.env` (copy `oidc.env.example`; it is
gitignored). On a deployed box, they are written into `/etc/wiab/wiab.env` by Terraform from the
`oidc_issuer`, `oidc_client_id` and `oidc_client_secret` variables — setting all three is what
flips `WIAB_AUTH_OIDC_ENABLED` on.

## The `email_verified` requirement

**This is the failure most Entra setups hit.**

The enterprise connection is configured with `require_email_verified = true`. If the ID token
does not carry an `email_verified: true` claim, the login is rejected outright with *"the
identity provider did not supply a verified email"* — it does not fall back to creating an
unlinked account.

Entra frequently omits `email_verified`. If it does, sign-in fails until the claim is emitted —
configure the claim in the app registration's token configuration, or use an IdP that sends it.

This is deliberate. The connection also has `auto_link_verified_email = true`, meaning a
successful login adopts whatever local account matches the asserted address. Trusting an
*unverified* address alongside auto-linking would let an IdP that can be induced to emit a
victim's address hand over that account, including a password-based Owner. The two settings are
a pair.

> The address itself does not have to arrive in `email`. Entra typically sends it as
> `preferred_username` (the UPN), and the adapter falls back to that claim. It is
> `email_verified` that must be present, not `email`.

Pre-provision users by their **UPN/email** so a real login matches an existing account.

## Per-environment credentials

Every environment that completes a sign-in performs the server-side code exchange, so each needs
valid credentials. Two options:

- Add that environment's HTTPS redirect URI to the same registration (**Manage →
  Authentication**) and reuse the client ID + secret.
- **Recommended beyond throwaway environments:** create a separate registration per environment,
  so dev and the deployed box don't share a secret and can be rotated or revoked independently.

## Verifying

Start the backend and check the startup log for the enterprise connection. Then visit
`/api/auth/oidc/enterprise/start` — you should be redirected to the Microsoft sign-in page and
land back on the box signed in.

Common failures:

- **Redirect URI mismatch** (Entra error `AADSTS50011`) — the registered URI differs from
  `<WIAB_BASE_URL>/api/auth/oidc/enterprise/callback`. Check for a trailing slash, `http` vs
  `https`, or a `WIAB_BASE_URL` that doesn't match the host the browser used.
- **"did not supply a verified email"** — see the `email_verified` section above.
- **Nothing happens / the connection is absent** — `WIAB_AUTH_OIDC_ENABLED` is not truthy, or
  one of the three credential variables is empty.
