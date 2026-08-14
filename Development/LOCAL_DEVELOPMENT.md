# Local development stack

One command brings up Postgres, the backend, the frontend, a mail catcher and a mock OIDC
provider:

```sh
docker compose -f dev/local/docker-compose.yml up --build
```

Run it from the workspace root — the directory holding `backend/`, `frontend/` and `dev/`.

**[`dev/local/README.md`](../../dev/local/README.md) is the reference** and is maintained next to
the compose file: service URLs, start/stop commands, where data is bind-mounted, how to reset the
database and git repos safely, and how to grab the bootstrap token from the logs. Read that
first. This page only covers the parts that interact with the operator-facing setup documented
elsewhere.

Two things worth knowing before the first run: the backend image compiles Rust plus several
cmake-built native dependencies and can take tens of minutes, and WebRTC media does not traverse
Docker's bridge network, so meetings connect but carry no audio.

## Email in local development

The stack bundles **Mailpit**, which accepts every message and shows it at
<http://localhost:8025>. The compose file already points the SMTP settings at it
(`WIAB_SMTP_HOST=mailpit`, `WIAB_SMTP_PORT=1025`), so the only thing needed to use it is to
select the SMTP transport:

```sh
WIAB_EMAIL_PROVIDER=smtp
```

Put that in `dev/local/oidc.env` (copy `oidc.env.example`; gitignored).

> Set `WIAB_EMAIL_FROM` in `oidc.env` as well, not in the compose file. Compose's `environment:`
> block overrides `env_file:`, so a value hardcoded there could not be changed per developer.
> This is why the compose file deliberately leaves `WIAB_EMAIL_FROM` unset.

You can also configure no provider at all. Reset and invite links are then written to the backend
log, which is often faster than opening a mail UI:

```sh
docker compose -f dev/local/docker-compose.yml logs backend | grep -i "reset\|invite"
```

For the delivery paths themselves, and setting up a real provider, see
[`../Installation/EMAIL_DELIVERY.md`](../Installation/EMAIL_DELIVERY.md).

## Enterprise SSO against the mock provider

The stack includes a mock OIDC provider on <http://localhost:9090>, off unless you enable it.

There is a hostname subtlety: the mock's discovery document reports whatever host the request
arrived on, so the browser and the backend must reach it under the **same** hostname. Running the
backend inside compose while the browser uses `localhost` does not satisfy that. Real identity
providers have no such problem.

To test against a real provider instead, see
[`../Installation/MICROSOFT_ENTRA.md`](../Installation/MICROSOFT_ENTRA.md) and
[`../Installation/GOOGLE_OAUTH.md`](../Installation/GOOGLE_OAUTH.md). Credentials go in
`dev/local/oidc.env`, and the redirect URIs registered with the provider must use
`http://localhost:3000` to match `WIAB_BASE_URL`.

## Backend on the host

To iterate on backend code without rebuilding the container, `backend/scripts/run-pg.sh` starts
the same Postgres in Docker and runs the backend with `cargo run` on the host.
