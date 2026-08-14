# Environment variables

Every environment variable read anywhere in the workinabox system, by component.

Verified against the source on 2026-08-14. Where a doc comment and the code disagreed, the code
wins. Each section names the file that resolves the variables, which is where to check when this
page falls behind.

**Boolean values.** The backend treats `1`, `true`, `yes` and `on` (case-insensitive) as true.
Anything else — including unset, empty, or `false` — is false.

## Contents

- [Backend](#backend) — process, VM runtime, auth, email, models, media, messaging, dev seeding
- [Agent team (`sw-dev-team`)](#agent-team-sw-dev-team)
- [In-guest agent (`wiab-agent`)](#in-guest-agent-wiab-agent)
- [Frontend](#frontend)
- [Website](#website)
- [Dev CLI](#dev-cli)
- [Infrastructure / provisioning](#infrastructure--provisioning)
- [CI](#ci)

---

## Backend

Resolved once at startup in [`backend/src/config.rs`](../../backend/src/config.rs), with
sub-configs in `backend/crates/wiab-inf/src/`. Nothing else in the backend reads the environment.

On a deployed box these live in `/etc/wiab/wiab.env`, read by the `wiab` systemd unit.

### Process and serving

| Variable | Default | Notes |
| --- | --- | --- |
| `WIAB_PERSISTENCE` | `postgres` | `postgres` or `memory`. Also settable as `--persistence`. |
| `DATABASE_URL` | `postgres://wiab:wiab@localhost:5432/wiab` | Postgres connection URL. Also settable as `--database-url`. |
| `RUST_LOG` | `wiab=info,tower_http=info` | Tracing filter. |
| `WIAB_BASE_URL` | `http://localhost:3000` | Public origin. Drives cookie `Secure`, email links, and OIDC redirect URIs. |
| `WIAB_GIT_ROOT` | a temp dir (`$TMPDIR/wiab-git`) | Directory hosting git repos. **Set this in production** — the default does not survive a reboot. |
| `WIAB_GIT_SSH_ADDR` | `0.0.0.0:2222` | Bind address of the git SSH transport. |
| `WIAB_GIT_SSH_HOST_KEY` | unset | Path to a persistent SSH host key. **Required for non-local `WIAB_BASE_URL`** — startup fails without it, since a regenerated host key trips every client's known-hosts check. |
| `WIAB_TLS_CERT` / `WIAB_TLS_KEY` | unset | PEM cert/key paths. With both unset the backend generates a self-signed certificate. |
| `WIAB_BOOTSTRAP_TOKEN_FILE` | unset | Path to write the short-lived bootstrap token (mode 0600). Without it the token is only logged. |
| `WIAB_VERSION` | package version | **Compile-time**, not runtime (`option_env!`). Release builds inject the git tag; otherwise `CARGO_PKG_VERSION`. |

### Authentication

Setup runbooks for the values in this section:
[`../Installation/GOOGLE_OAUTH.md`](../Installation/GOOGLE_OAUTH.md) and
[`../Installation/MICROSOFT_ENTRA.md`](../Installation/MICROSOFT_ENTRA.md).

| Variable | Default | Notes |
| --- | --- | --- |
| `WIAB_AUTH_LOCAL_SIGNUP` | off | Enables self-service signup. |
| `WIAB_AUTH_GOOGLE_ENABLED` | off | Enables the `google` federation connection. |
| `WIAB_GOOGLE_CLIENT_ID` | empty | |
| `WIAB_GOOGLE_CLIENT_SECRET` | empty | |
| `WIAB_AUTH_OIDC_ENABLED` | off | Enables the `enterprise` federation connection. |
| `WIAB_OIDC_ISSUER` | empty | e.g. `https://login.microsoftonline.com/<tenant-id>/v2.0`. |
| `WIAB_OIDC_CLIENT_ID` | empty | |
| `WIAB_OIDC_CLIENT_SECRET` | empty | |

Credentials default to empty rather than failing, so a connection whose `*_ENABLED` flag is on
but whose credentials are missing simply does not work. Both connections require a verified
email claim.

### Email

| Variable | Default | Notes |
| --- | --- | --- |
| `WIAB_EMAIL_PROVIDER` | `resend` | `resend` or `smtp`. An unknown value logs mail instead of sending it. |
| `WIAB_EMAIL_FROM` | `no-reply@workinabox.local` | Falls back to `WIAB_SMTP_FROM` when unset. |
| `WIAB_SMTP_FROM` | — | Legacy alias, only consulted when `WIAB_EMAIL_FROM` is unset. |
| `RESEND_API_KEY` | unset | Required by the `resend` provider; without it, delivery degrades to logging. |
| `WIAB_SMTP_HOST` | unset | Required by the `smtp` provider; without it, delivery degrades to logging. |
| `WIAB_SMTP_PORT` | `587` | |
| `WIAB_SMTP_USER` | unset | |
| `WIAB_SMTP_PASSWORD` | unset | |
| `WIAB_SMTP_TLS` | off | |

Setup: [`../Installation/EMAIL_DELIVERY.md`](../Installation/EMAIL_DELIVERY.md).

### VM runtime

The backend runs agents in Firecracker microVMs when `WIAB_FIRECRACKER_ENABLED` is truthy **and**
`/dev/kvm` is present; otherwise it falls back to the Docker backend.

| Variable | Default | Notes |
| --- | --- | --- |
| `WIAB_FIRECRACKER_ENABLED` | off | Half the decision; `/dev/kvm` is the other half. |
| `WIAB_AGENT_RUNTIME` | `rust` | `rust` (the `wiab-agent` binary) or `langgraph` (the Python dev team). An unrecognised value warns and uses `rust`. |
| `WIAB_MODEL_ENDPOINT` | empty | Model endpoint handed to the guest. Read by both runtimes. |

Docker backend ([`docker_runtime.rs`](../../backend/crates/wiab-inf/src/docker_runtime.rs)):

| Variable | Default | Notes |
| --- | --- | --- |
| `WIAB_DOCKER_IMAGE_PREFIX` | per runtime | `wiab-agent-` for `rust`, `wiab-team-` for `langgraph`. |
| `WIAB_DOCKER_IMAGE_TAG` | `latest` | |
| `WIAB_DOCKER_NETWORK` | empty | |
| `WIAB_DOCKER_STOP_TIMEOUT` | `10` | Seconds. |
| `WIAB_WORKSPACE_MIB` | `2048` | Writable workspace size. Only the `langgraph` runtime needs one. |

Firecracker backend ([`firecracker_runtime.rs`](../../backend/crates/wiab-inf/src/firecracker_runtime.rs)):

| Variable | Default |
| --- | --- |
| `WIAB_MICROVM_SUBNET` | `172.16.0.0/24` |
| `WIAB_IMAGES_DIR` | `/var/lib/wiab/images` |
| `WIAB_VMS_DIR` | `/var/lib/wiab/vms` |
| `WIAB_AGENT_DIR` | `/var/lib/wiab/agent` |
| `WIAB_JAIL_BASE` | `/srv/jailer` |
| `WIAB_JAILER_BIN` | `/usr/local/bin/jailer` |
| `WIAB_FIRECRACKER_BIN` | `/usr/local/bin/firecracker` |
| `WIAB_VMM_CTL_BIN` | `/usr/local/bin/wiab-vmm-ctl` |
| `WIAB_VM_OVERLAY_MIB` | `2048` |
| `WIAB_JAIL_UID` | `123` |
| `WIAB_JAIL_GID` | `100` |

### Local models

Model files are read from `<data dir>/models/<filename>`. An enabled model whose file is missing
aborts startup.

| Variable | Default | Notes |
| --- | --- | --- |
| `WIAB_DATA_DIR` | `$HOME/.local/share/wiab` | Base data directory. Startup fails if this is unset and `HOME` is unavailable. |
| `HOME` | — | Only used for the `WIAB_DATA_DIR` fallback above. |

Llama (meeting intelligence) — the whole group is read only when enabled:

| Variable | Default | Notes |
| --- | --- | --- |
| `WIAB_LLAMA_ENABLED` | off | |
| `WIAB_LLAMA_MODEL_FILE` | — | **Required** when enabled. Filename within the models dir. |
| `WIAB_LLAMA_CONTEXT_TOKENS` | `4096` | Must be > 0. |
| `WIAB_LLAMA_MAX_REPLY_TOKENS` | `128` | Must be > 0. |
| `WIAB_LLAMA_MAX_MINUTES_TOKENS` | `512` | |
| `WIAB_LLAMA_THREADS` | detected from the host | |
| `WIAB_LLAMA_N_GPU_LAYERS` | `0` | |
| `WIAB_LLAMA_CHAT_TEMPLATE` | unset | Named chat template override. |

Whisper (speech-to-text):

| Variable | Default | Notes |
| --- | --- | --- |
| `WIAB_WHISPER_ENABLED` | off | |
| `WIAB_WHISPER_MODEL_FILE` | — | **Required** when enabled. |
| `WIAB_STT_LANGUAGE` | unset (auto-detect) | |
| `WIAB_STT_THREADS` | `4` | Values ≤ 0 fall back to the default. |

### Media (WebRTC / mediasoup)

| Variable | Default | Notes |
| --- | --- | --- |
| `WIAB_MEDIASOUP_LISTEN_IP` | `0.0.0.0` | An unparseable value falls back to the default. |
| `WIAB_MEDIASOUP_ANNOUNCED_ADDRESS` | `10.0.2.2` | Address announced to clients — the public/NAT address in a real deployment. |
| `WIAB_MEDIASOUP_MIN_PORT` | `40000` | |
| `WIAB_MEDIASOUP_MAX_PORT` | `40999` | Startup fails if min > max. Must match the range the host firewall opens. |

### Messaging

| Variable | Default | Notes |
| --- | --- | --- |
| `WIAB_NATS_ENABLED` | off | With it off, the backend serves HTTP and simply does not publish. |
| `WIAB_NATS_URL` | `nats://127.0.0.1:4222` | Only read when enabled. |

### Dev seeding

Conveniences for local development. Do not set these on a shared deployment.

| Variable | Default | Notes |
| --- | --- | --- |
| `WIAB_DEV_OWNER_PASSWORD` | none | Password for the seeded bootstrap Owner. **No baked-in default** — a non-local `WIAB_BASE_URL` refuses to start without it, rather than shipping a published credential. |
| `WIAB_DEV_SSO_OWNER_EMAIL` | unset | Pre-provisions this address as an Active Owner of `O-1`. The whole SSO-seeding block is a no-op unless this is set. Match the IdP's UPN exactly. |
| `WIAB_DEV_SSO_OWNER_NAME` | the email | |
| `WIAB_DEV_SSO_OWNER_PASSWORD` | unset | Local password fallback for that account. |

---

## Agent team (`sw-dev-team`)

The Python/LangGraph dev team, used when `WIAB_AGENT_RUNTIME=langgraph`. Resolved in
`sw-dev-team/src/wiab_team/config.py`; `uv run wiab-team validate-config` checks the environment
without running anything.

**[`sw-dev-team/docs/ENV.md`](../../sw-dev-team/docs/ENV.md) is the authoritative reference** and
carries the rationale behind each option. Summarised here for completeness.

The backend forwards its own environment into a LangGraph guest **by prefix**: everything
starting with `WIAB_TEAM_`, plus `ANTHROPIC_API_KEY`. Adding an option to the team needs no
backend change.

### Model

| Variable | Default | Notes |
| --- | --- | --- |
| `ANTHROPIC_API_KEY` | — | Required unless the tool provider is `stub`. |
| `WIAB_TEAM_MODEL` | `claude-opus-4-8` | Default model for every role. |
| `WIAB_TEAM_EFFORT` | unset | `low`, `medium`, `high`, `xhigh`, `max`. |
| `WIAB_TEAM_ARCHITECT_MODEL` / `WIAB_TEAM_ARCHITECT_EFFORT` | the defaults above | |
| `WIAB_TEAM_DEV_MODEL` / `WIAB_TEAM_DEV_EFFORT` | the defaults above | |
| `WIAB_TEAM_TESTER_MODEL` / `WIAB_TEAM_TESTER_EFFORT` | the defaults above | |

### Runtime

| Variable | Default | Notes |
| --- | --- | --- |
| `WIAB_TEAM_TOOL_PROVIDER` | `claude_sdk` | `claude_sdk` or `stub`. |
| `WIAB_TEAM_WORKSPACE` | `/workspace` | Must be writable — worktrees live here. |
| `WIAB_TEAM_DEV_COUNT` | `3` | Fan-out, 1–8. A payload's `max_devs` overrides it. |
| `WIAB_TEAM_MAX_REPAIR_ROUNDS` | `2` | Retries after a failing test suite, 0–10. |
| `WIAB_TEAM_MAX_CONFLICT_ROUNDS` | `1` | Retries after a merge conflict, 0–5. |
| `WIAB_TEAM_CLAUDE_CLI` | unset | Path to the `claude` CLI when it is not on `PATH`. |
| `WIAB_TEAM_PROMPTS_DIR` | `/opt/wiab-team-prompts` in the container image | Directory of `<name>.md` prompt overrides: `architect`, `dev`, `tester_scaffold`, `tester_final`. Unrecognised filenames warn at startup. |
| `WIAB_TEAM_PAYLOAD` | unset | Read by the container entrypoint, not the package: naming an existing file switches from `work` (poll the board) to a one-shot `run`. |

### Team worker

Read by `wiab-team work` only. The first five are required — the backend sets all of them itself
when it starts a team, so you set them by hand only to run a team outside the backend.

| Variable | Default | Notes |
| --- | --- | --- |
| `WIAB_TEAM_API_URL` | — | **Required.** Backend base URL. |
| `WIAB_TEAM_TEAM_ID` | — | **Required.** This team's `TM-<n>` id. |
| `WIAB_TEAM_BOARD_ID` | — | **Required.** The `B-<n>` board to pull from. |
| `WIAB_TEAM_REPO_REMOTE` | — | **Required.** Clone URL, `<api>/repos/R-<n>.git`. |
| `WIAB_TEAM_API_TOKEN` | — | **Required.** Bearer token for the backend. |
| `WIAB_TEAM_BASE_BRANCH` | `main` | Branch each issue is cut from. |
| `WIAB_TEAM_POLL_INTERVAL_SECONDS` | `10` | 0.1–3600. Also how quickly a pause is noticed. |
| `WIAB_TEAM_STATUS_PORT` | `8081` | Read-only status endpoint (`/health`, `/status`). `0` disables it. |
| `WIAB_TEAM_API_CA_PEM` | unset | The backend's TLS certificate, in PEM. **Without it nothing in the container trusts a self-signed backend and every request fails.** |
| `GIT_SSL_CAINFO` | set by the team | Not an input: pointed at `<workspace>/backend-ca.pem`, written from `WIAB_TEAM_API_CA_PEM`, because git reads a certificate from a file. |

### Delivery, observability, checkpointing

| Variable | Default | Notes |
| --- | --- | --- |
| `WIAB_TEAM_GIT_TOKEN` | unset | Required to push or open a PR. For a backend-started team it is the same token as `WIAB_TEAM_API_TOKEN`. |
| `WIAB_TEAM_LOG_LEVEL` | `INFO` | `DEBUG`/`INFO`/`WARNING`/`ERROR`/`CRITICAL`. |
| `WIAB_TEAM_LOG_JSON` | `1` | `0` for human-readable console output. Keep JSON in production — on the Docker path stdout is the only observability surface. |
| `WIAB_TEAM_CHECKPOINT_DSN` | unset | Postgres DSN, confined to the `wiab_team` schema. **Pausing mid-issue requires this**; without it a team can only be paused between issues. |

---

## In-guest agent (`wiab-agent`)

The Rust agent binary, running inside the sandbox. Reads its config from the agent drive.

| Variable | Default | Notes |
| --- | --- | --- |
| `WIAB_AGENT_ID` | `A-?` | |
| `WIAB_MODEL_ENDPOINT` | empty | |

---

## Frontend

Vite; `VITE_*` variables are **baked into the bundle at build time**, not read at runtime. Local
values go in `frontend/.env.local` (see `frontend/.env.example`).

| Variable | Default | Notes |
| --- | --- | --- |
| `VITE_USE_STUB` | `true` in the example | `true` serves the Works list from the in-app stub, no backend needed. |
| `VITE_API_BASE_URL` | `/api` | The Vite dev server proxies `/api` to `http://localhost:8080`. |
| `VITE_APP_VERSION` | — | Injected at build time. |

---

## Website

Vite; same build-time baking. See `website/.env.example`. Firebase web config values are public
by design.

| Variable | Notes |
| --- | --- |
| `VITE_FIREBASE_API_KEY` | From Firebase console → Project settings. |
| `VITE_FIREBASE_AUTH_DOMAIN` | |
| `VITE_FIREBASE_PROJECT_ID` | |
| `VITE_FIREBASE_APP_ID` | |
| `VITE_FIREBASE_STORAGE_BUCKET` | |
| `VITE_FIREBASE_MESSAGING_SENDER_ID` | |
| `VITE_LAUNCHED` | `false` shows the "Coming soon" holding page; allowlisted users can still sign in to preview. `true` makes the site fully public. |
| `VITE_PREVIEW_ALLOWLIST` | Comma-separated emails allowed to preview while `VITE_LAUNCHED=false`. |
| `VITE_GA_MEASUREMENT_ID` | GA4 id (`G-XXXXXXXXXX`). Blank disables analytics. GA is consent-gated regardless. |
| `VITE_APP_VERSION` | Injected at build time. |

The mobile app (`app/`) reads no environment variables.

---

## Dev CLI

The Ratatui dev tool in `dev/`.

| Variable | Notes |
| --- | --- |
| `GITHUB_WORKINABOX_TOKEN` | GitHub token for the workinabox repos. |
| `GH_TOKEN` | Fallback GitHub token. |

---

## Infrastructure / provisioning

Terraform-driven. These are not read by application code — they configure the host and are the
source of the backend variables above. For the Terraform side, see
[`TERRAFORM_TFVARS.md`](TERRAFORM_TFVARS.md).

Two files on the box, both under `/etc/wiab`:

- **`provision.env`** — written by cloud-init from Terraform variables; read by `provision.sh`
  and `wiab-deploy`. Deliberately carries no secrets: cloud-init user-data is retrievable by
  anyone with pool access and is stored rendered in Terraform state.
- **`wiab.env`** — the backend's systemd `EnvironmentFile` (mode 0640 `root:wiab`). Holds the DB
  password, the Resend key and the OIDC client secrets, pushed over SSH after boot rather than
  through cloud-init.

### `provision.env`

| Variable | Terraform source | Notes |
| --- | --- | --- |
| `WIAB_DOMAIN` | `var.domain` | |
| `WIAB_LETSENCRYPT_EMAIL` | `var.letsencrypt_email` | |
| `WIAB_ANNOUNCED_ADDRESS` | `var.announced_address` | Becomes `WIAB_MEDIASOUP_ANNOUNCED_ADDRESS`. |
| `WIAB_BACKEND_REPO` / `WIAB_FRONTEND_REPO` | `var.backend_repo` / `var.frontend_repo` | Release artifact sources. |
| `WIAB_BACKEND_VERSION` / `WIAB_FRONTEND_VERSION` | `var.backend_version` / `var.frontend_version` | |
| `WIAB_DATA_DIR` | `var.wiab_data_dir` (`/var/lib/wiab`) | |
| `WIAB_FC_KERNEL_URL` / `WIAB_FC_ROOTFS_URL` | `var.fc_test_kernel_url` / `var.fc_test_rootfs_url` | |
| `WIAB_IMAGE_FILES` | `var.vm_image_files` | Space-separated. |
| `WIAB_IMAGES_VERSION` | `var.vm_images_version` | Bump to re-fetch after rebuilding an image. |
| `WIAB_IMAGES_URL` | `var.wiab_images_url` | Azure blob URL **with an embedded SAS token**. Pushed over SSH, not cloud-init. |
| `WIAB_MODELS_URL` | `var.wiab_models_url` | Same shape. Also used by `backend/scripts/fetch-models.sh` locally. |
| `WIAB_MICROVM_SUBNET` | `var.microvm_subnet` (`172.16.0.0/24`) | |
| `WIAB_SSH_ADMIN_CIDR` | `var.ssh_admin_cidr` | |
| `WIAB_SSH_PUBKEY` | `var.ssh_pubkey`, falling back to `var.ssh_authorized_key` | Baked into guest images. |
| `WIAB_MEDIASOUP_MIN_PORT` / `WIAB_MEDIASOUP_MAX_PORT` | `var.rtc_min_port` / `var.rtc_max_port` | The firewall opens exactly this range. |
| `WIAB_<ROLE>_ENABLED` / `WIAB_<ROLE>_MODEL_FILE` | `var.models` | Generated per role — `LLAMA` and `WHISPER` produce the backend variables documented above. |

### Deploy state

Recorded in `/etc/wiab/versions` by `wiab-deploy` so unchanged artifacts are skipped. Not inputs
you set.

| Variable | Notes |
| --- | --- |
| `WIAB_MODELS_FINGERPRINT` | Fingerprint of the deployed model set. |
| `WIAB_IMAGES_VERSION` | Last deployed image version. |
| `WIAB_AGENT_VERSION` | Last deployed agent image tag. |

---

## CI

GitHub Actions secrets and repository variables.

| Name | Repo | Notes |
| --- | --- | --- |
| `GITHUB_TOKEN` | all | Provided by Actions. |
| `WIAB_IMAGES_URL` | `iac` | Secret. Upload target for built VM images. |
| `WIAB_GUEST_SSH_PUBKEY` | `iac` | Secret. Build arg baked into guest images. |
| `FIREBASE_SERVICE_ACCOUNT` | `website` | Secret. Deploy credential. |
| `FIREBASE_PROJECT_ID` | `website` | Variable. |
| `VITE_FIREBASE_*` | `website` | Secrets, baked into the bundle at build time. |
| `VITE_PREVIEW_ALLOWLIST` | `website` | Secret. |
| `VITE_LAUNCHED`, `VITE_GA_MEASUREMENT_ID` | `website` | Variables. |
| `WIAB_VERSION` | `backend` | Injected at build time as the release git tag. |
