# Installing email delivery

The backend sends transactional email: invitations, password resets, and address verification.
Until a provider is configured, those messages are **written to the server log instead of sent**
— which is fine for development and silently useless in production.

Two transports are supported, selected by `WIAB_EMAIL_PROVIDER`: `resend` (default, an HTTPS API)
and `smtp`.

## Resend (default)

One authenticated HTTPS call per email — no SMTP, no Basic auth.

1. Create an account at [resend.com](https://resend.com).
2. **Verify a sending domain** (Domains → Add Domain), which means adding the DNS records Resend
   gives you. Delivery from an unverified domain will not work.
3. Create an API key.
4. Configure the backend:

   | Variable | Value |
   | --- | --- |
   | `WIAB_EMAIL_PROVIDER` | `resend` (the default; may be omitted) |
   | `RESEND_API_KEY` | the API key |
   | `WIAB_EMAIL_FROM` | an address on your verified domain |

**For a quick test without a domain**, use `onboarding@resend.dev` as `WIAB_EMAIL_FROM`. Resend
only delivers from that address to your own Resend account address, so it proves the wiring works
and nothing more.

If `RESEND_API_KEY` is unset, startup logs `email delivery: logging only` and every message goes
to the log.

## SMTP

For an existing mail relay, or the local Mailpit catcher.

| Variable | Default | Notes |
| --- | --- | --- |
| `WIAB_EMAIL_PROVIDER` | — | Must be `smtp`. |
| `WIAB_SMTP_HOST` | — | Required. Without it, delivery falls back to logging. |
| `WIAB_SMTP_PORT` | `587` | |
| `WIAB_SMTP_USER` | unset | Optional. |
| `WIAB_SMTP_PASSWORD` | unset | Optional. |
| `WIAB_SMTP_TLS` | off | Leave off for a local Mailpit on `:1025`. |
| `WIAB_EMAIL_FROM` | `no-reply@workinabox.local` | |

**Microsoft 365 is a poor fit** — it is retiring Basic-auth SMTP. Gmail (with an app password),
SendGrid, or Amazon SES are better choices.

An SMTP configuration the client rejects as invalid logs the error and falls back to logging
rather than failing startup.

## Deployed boxes

Terraform writes `WIAB_EMAIL_PROVIDER=resend`, `WIAB_EMAIL_FROM` and `RESEND_API_KEY` into
`/etc/wiab/wiab.env` from the `email_from` and `resend_api_key` variables. The deployed path is
Resend-only today — using SMTP on a real box means editing `wiab.env` by hand, which the next
deploy will overwrite.

## Verifying

Watch the startup log. The backend states which transport it chose, and the message is
unambiguous:

- `email delivery: Resend (from <address>)`
- `email delivery: SMTP <host>:<port>`
- `email delivery: logging only (set RESEND_API_KEY to send via Resend)`
- `email delivery: logging only (set WIAB_SMTP_HOST to send via SMTP)`
- `email delivery: unknown WIAB_EMAIL_PROVIDER '<value>', logging only`

Then trigger a real message — invite a user, or request a password reset — and confirm it
arrives. A misspelled `WIAB_EMAIL_PROVIDER` does not fail startup; it produces the last line
above and silently logs mail, so check for it.
