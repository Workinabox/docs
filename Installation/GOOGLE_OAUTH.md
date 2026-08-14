# Installing Google social login

How to register an OAuth client in Google Cloud and point a backend at it, so the sign-in page
offers "Continue with Google".

This is the *consumer* social login path. For a company's own identity provider, see
[`MICROSOFT_ENTRA.md`](MICROSOFT_ENTRA.md) — they are the same relying-party code with different
configuration, and a box can run both at once.

## Before you start

You need a Google Cloud project you can create OAuth clients in, and the public base URL of the
box (`WIAB_BASE_URL`) — the redirect URI is derived from it and must match byte-for-byte.

## Create the OAuth client

1. Go to the Google Cloud **Clients** page: <https://console.cloud.google.com/auth/clients>.
   Create or select a project.
2. First time only, configure the **OAuth consent screen / Branding**: app name, support email,
   and **User type = External**.

   While the app's publishing status is **Testing**, only Google accounts listed under **Test
   users** can sign in. **Add your own account there**, or publish the app for general
   availability — otherwise your first login fails with an access-denied error that looks like a
   misconfiguration.
3. **Create Client** → application type **Web application**.
4. Under **Authorized redirect URIs**, add the callback for each environment, byte-for-byte:

   ```text
   ${WIAB_BASE_URL}/api/auth/oidc/google/callback
   ```

   - local: `http://localhost:3000/api/auth/oidc/google/callback`
   - deployed: `https://<host>/api/auth/oidc/google/callback` — must be HTTPS; plain `http` is
     accepted only for `localhost`.

   The path is fixed. The backend builds it as `<WIAB_BASE_URL>/api/auth/oidc/google/callback`,
   where `google` is the connection slug.
5. Copy the **Client ID** and **Client secret**.

## Configure the backend

| Google value | Environment variable |
| --- | --- |
| Client ID | `WIAB_GOOGLE_CLIENT_ID` |
| Client secret | `WIAB_GOOGLE_CLIENT_SECRET` |
| — | `WIAB_AUTH_GOOGLE_ENABLED=true` |

The issuer is not configurable — it is hardcoded to `https://accounts.google.com`. The backend
requests the `openid`, `email` and `profile` scopes.

The connection is registered at startup only when `WIAB_AUTH_GOOGLE_ENABLED` is truthy
(`1`/`true`/`yes`/`on`). The two credentials default to empty, so a box with the flag on and no
credentials shows the button and fails the exchange.

Locally these go in `dev/local/oidc.env` (copy `oidc.env.example`; gitignored). On a deployed box
Terraform writes them into `/etc/wiab/wiab.env` from `google_client_id` and
`google_client_secret` — setting both is what flips `WIAB_AUTH_GOOGLE_ENABLED` on.

## Account linking behavior

**Google logins do not adopt existing local accounts.** The connection is configured with
`auto_link_verified_email = false`.

- A Google user with **no** local account is provisioned just-in-time and can sign in.
- A Google address that **matches an existing password account** is refused. The login returns an
  account-already-exists error rather than silently handing the Google identity control of that
  account.

This is a deliberate anti-takeover choice, and it differs from the enterprise connection, which
*does* auto-link (and therefore demands a verified email in exchange).

> There is currently **no link flow** to resolve the second case — no endpoint lets a signed-in
> user attach a Google identity to their existing account. A user in that position cannot use the
> Google button today; they sign in with their password instead.

Google reliably stamps `email_verified`, and the connection requires it. In practice this is not
something you configure.

## Per environment

Each environment that completes a sign-in performs the server-side code exchange, so each needs
working credentials. Either add every environment's redirect URI to the same client, or create a
separate client per environment so they can be rotated independently. The consent screen is
shared across the whole Google Cloud project either way; moving it from Testing to published
removes the test-user restriction everywhere.

## Verifying

Start the backend and confirm the "Continue with Google" button appears on the sign-in page
(driven by `/api/auth/config`). Then complete a login.

Common failures:

- **`redirect_uri_mismatch`** — the registered URI differs from
  `<WIAB_BASE_URL>/api/auth/oidc/google/callback`. Check for a trailing slash, `http` vs `https`,
  or a `WIAB_BASE_URL` that doesn't match the host the browser used.
- **Access denied for your own account** — the app is in Testing and you are not a test user.
- **Account-exists error** — the address already has a password account. See the linking section
  above.
- **No button** — `WIAB_AUTH_GOOGLE_ENABLED` is not truthy.
