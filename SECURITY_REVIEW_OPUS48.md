# Security Review — Workinabox

Date: 2026-07-04
Reviewer: automated multi-agent security audit (Claude Code)
Scope: all nine repositories in the `Workinabox` GitHub organisation
Method: full source read of every repo plus targeted static analysis; every
Critical and High finding was independently re-verified against the source.

## How to read this document

Findings are grouped by severity (Critical first) and given a stable ID
(`C1`, `H3`, `M7`, …) so they can be referenced in tickets. Each finding names
the repository it lives in, the exact `file:line`, the concrete issue, the
attacker impact, and a recommended fix. A consolidated remediation roadmap is at
the end, followed by the (substantial) list of things this codebase already does
well.

Severity meaning:

- **Critical** — remotely exploitable now, leads to full compromise or total
  data disclosure with no authentication.
- **High** — serious exposure; either exploitable with modest preconditions or a
  compromise of a key trust boundary.
- **Medium** — real weakness that materially lowers the security bar but needs a
  precondition, is bounded in blast radius, or is defence-in-depth.
- **Low** — minor issue, hardening gap, or latent risk that is not currently
  exploitable.
- **Info** — observations, non-issues clarified, and things to keep in view.

## Executive summary

The platform is, at the crypto and application-service layer, **well
engineered** — Argon2id password hashing, CSPRNG tokens stored only as hashes,
double-submit CSRF, parameterized SQL throughout, no command injection in the
git/VM shell-outs, strict `RepoId`/branch-name validation, public-key-only SSH,
PKCE + nonce + JWKS-validated OIDC, jailer-isolated Firecracker microVMs, and
SHA-256-verified release artifacts with rollback. There are no committed secrets
in any repository's git history.

The dominant risk is a **single systemic gap**: the backend HTTP API applies
authorization per-handler rather than through a global middleware, and the
majority of handlers were never given a guard. As a result an unauthenticated
network client can read every private repository's source and mutate every
tenant's data. This is the headline issue (C1/C2) and it dwarfs everything else.
The second theme is **operational secret hygiene** on the infrastructure side
(local Terraform state and a world-readable env file holding live secrets), and
the third is **missing rate limiting / brute-force protection** across the auth
surface.

### Finding counts by severity

| Severity | Count |
|----------|:-----:|
| Critical | 3 |
| High | 9 |
| Medium | 17 |
| Low | 20 |
| Info | 10 |

### Top risks to fix first

1. **C1/C2 — Unauthenticated REST API.** Add a global auth middleware and
   per-resource authorization to every backend handler. Nothing else matters as
   much; today the API is effectively public.
2. **C3 — Plaintext secrets in local Terraform state and `tfvars`.** Move to an
   encrypted remote backend and rotate every secret those files contain.
3. **H1 — `owner/owner` default admin plus plaintext token logged at boot.**
   Refuse a default password in production and stop logging credentials.
4. **H5/H6/H7 — Infrastructure exposure.** Xen Orchestra TLS verification is
   disabled, `wiab.env` is world-readable, and untrusted microVMs can pivot to
   the host and LAN.

## Scope

| Repo | Stack | Role |
|------|-------|------|
| `backend` | Rust (axum, tokio, sqlx/tokio-postgres, russh, git2, Firecracker) | API, auth/identity (`authbox`), git hosting, SFU, agent sandbox |
| `frontend` | React + Vite + TypeScript, nginx, Docker | Web console (admin/management UI) |
| `website` | React + Vite + TypeScript, Firebase Hosting | Public marketing site |
| `app` | React Native (bare) | Mobile audio/meeting client |
| `iac` | Terraform (xenorchestra), cloud-init, shell | XCP-ng provisioning + deploy |
| `dev` | Rust CLI | `monitor` / `release` tooling |
| `docs` | Markdown | Planning + architecture docs (this report lives here) |
| `assets` | Images + a static handoff page | Visual identity |
| `.github` | Markdown | Org profile / architecture notes |

## Critical findings

### C1 — Unauthenticated state-changing REST endpoints across every tenant

- **Repo:** `backend`
- **Location:** `crates/wiab-inf/src/http_api.rs:34-153` (router; the only layer
  is `csrf_guard` at `:152`). Unguarded handlers verified by reading their
  bodies include `create_meeting` (`:181`), `create_organization` (`:205`),
  `update_organization` (`:232`), `create_project` (`:263`),
  `update_project` (`:294`), `create_agent` (`:325`), `update_agent` (`:374`),
  `activate_agent` (`:390`), `deactivate_agent` (`:405`), `create_board`
  (`:435`), `update_board` (`:466`), `update_repo` (`:538`), `create_pipeline`
  (`:1593`), `update_pipeline` (`:1624`), `create_work` (`:1655`),
  `update_work` (`:1686`), `add_done` (`:1702`), `fulfill_done`/`unfulfill_done`.
- **Issue:** Authorization is enforced per-handler, and only a minority of
  handlers (`create_repo`, `create_commit`, `set_repo_visibility`, the
  user/token/SSH-key/role endpoints, and the git-HTTP path) actually call
  `authenticate`/`require_*`. The rest take only `State`/`Path`/`Json`, never
  receive the request headers, and call their application service directly. The
  `csrf_guard` layer is not an authentication gate — it only acts when a session
  cookie is present and resolvable, so an anonymous request with no cookie passes
  straight through.
- **Impact:** Any unauthenticated network client can create, rename, or mutate
  organizations, projects, boards, pipelines, works, and agents for any tenant.
  The most severe instance is `create_agent` (`:340-354`), which provisions a new
  user identity and grants it `Role::Write` on the target org — an anonymous
  attacker can inflate the org's privileged identity set at will, and on a KVM
  host `activate_agent` will boot a microVM. This is a full multi-tenant
  integrity breach. Both backend review passes and a direct re-read of the router
  and handler bodies confirm it.
- **Recommendation:** Enforce authentication centrally with a
  `middleware::from_fn_with_state` layer (sibling to `csrf_guard`) that resolves
  identity for every route except an explicit public allow-list (`/health`,
  `/auth/*`, the git smart-HTTP endpoints, the OIDC callback). Then add a
  per-resource `require_org_role` / repo-role / ownership check to each handler,
  mirroring `create_repo` (`:...` calls `require_org_role`). Fail closed: a route
  with no explicit guard must be unreachable, not open.

### C2 — Unauthenticated read of any private repository's contents and history

- **Repo:** `backend`
- **Location:** `crates/wiab-inf/src/http_api.rs` — `read_repo_file` (`:599`),
  `list_repo_files` (`:581`), `list_repo_commits` (`:619`), `list_branches`
  (`:569`), `get_repo` (`:523`), `list_repos`; backing logic
  `crates/wiab-app/src/repo_application_service.rs:171-206` performs no
  visibility check.
- **Issue:** The git smart-HTTP transport correctly gates reads by repository
  `Visibility` and requires a token for private repos
  (`git_http.rs:219-255`). The parallel REST "browse" endpoints do not: they call
  the repo service directly, and the service only checks that the repo exists —
  never its visibility or the caller's role. Verified by reading `read_repo_file`
  (takes only `State`/`Path`/`Query`, no auth).
- **Impact:** Anyone who can reach the API can read full file contents, commit
  history, branch lists, and metadata of every **private** repository, for
  example `GET /repos/R-1/branches/main/files/raw?path=.env`. This bypasses the
  exact confidentiality control the git transport enforces — a complete
  disclosure of hosted source and any secrets committed to it.
- **Recommendation:** Fixed by the same central auth middleware as C1, plus a
  visibility/role check on every repo read handler: resolve the caller, allow the
  anonymous branch only when `repo_visibility == Public`, otherwise require
  `authorization_service.authorize(user, repo, Operation::Read, scope)`. Push the
  visibility check down into `RepoApplicationService` so no future caller can
  forget it.

### C3 — Plaintext infrastructure secrets in local Terraform state and tfvars, no encrypted backend

- **Repo:** `iac`
- **Location:** `terraform.tfstate`, `terraform.tfvars`, `versions.tf:1-10` (no
  `backend`/`cloud` block anywhere), `.gitignore:1-16`.
- **Issue:** There is no remote state backend. State lives only in
  `terraform.tfstate` on the operator's disk — unencrypted, unlocked. By field
  name (values were not extracted), it contains in cleartext: `db_password`,
  `google_client_secret` (twice), `oidc_client_secret` (twice), `resend_api_key`
  (twice), and the fully rendered cloud-init that embeds the DB password and the
  Azure blob **SAS tokens** for `WIAB_MODELS_URL` / `WIAB_IMAGES_URL`.
  `terraform.tfvars` holds the live populated values of `xoa_token` (Xen
  Orchestra API token), `db_password`, both SAS URLs, `resend_api_key`,
  `google_client_secret`, and `oidc_client_secret`. The `sensitive = true` flags
  only mask CLI output; they do not encrypt state, and provisioner `triggers`
  store values unmasked regardless.
- **Impact:** Anyone who reads these files (a backup, a stolen laptop, an
  accidental `git add -f`, a synced folder) obtains the hypervisor control-plane
  token, the OAuth/OIDC client secrets, the email-provider key, the DB password,
  and long-lived Azure SAS URLs — near-total compromise of the infrastructure and
  its identity integrations. Local state also means no locking (concurrent-apply
  corruption) and no audit trail. `.gitignore` does cover `*.tfstate*` /
  `*.tfvars` / `tfplan` (confirmed never committed), but that is the only thing
  preventing a leak, and one `-f` defeats it.
- **Recommendation:** Move to a remote backend with encryption-at-rest and state
  locking (Terraform Cloud, or S3+DynamoDB / `azurerm` blob with lock). Treat the
  on-disk `tfstate*` / `tfvars` as **already exposed** and rotate every secret
  they contain — XO token, DB password, Resend key, Google + OIDC client secrets
  — and regenerate both blob SAS tokens. In the interim set the files to `0600`.
  Keep secrets out of `triggers`/rendered templates; source them at apply time.

## High findings

### H1 — Default `owner/owner` admin and plaintext credentials logged at first boot

- **Repo:** `backend`
- **Location:** `src/bootstrap.rs:566-614` (`seed_owner`), default password at
  `:604-605`, log line at `:610-613`.
- **Issue:** On first boot (empty store) the backend seeds
  `owner@workinabox.local` with `Role::Owner` on `O-1`, a non-expiring bootstrap
  access token (`expires_at: None`), and password `"owner"` unless
  `WIAB_DEV_OWNER_PASSWORD` is set. It then logs at INFO both the plaintext
  password **and** the plaintext bootstrap token.
- **Impact:** Deployed without the override, `owner/owner` is a full-admin
  backdoor. Independently, the long-lived token and password are written to log
  storage/aggregation, readable by anyone with log access and outliving any
  session. Project notes indicate a live demo, so this is not hypothetical.
- **Recommendation:** Never log token plaintext or passwords. Refuse to start
  with the default owner password when `cookie_secure` is true / the base URL is
  non-local. Make the bootstrap token single-use or short-lived and surface it
  only on an operator channel, not the structured log.

### H2 — Unauthenticated real-time signaling WebSocket enables meeting hijack and eavesdropping

- **Repo:** `backend`
- **Location:** `crates/wiab-inf/src/http_api.rs:151` and `:1747-1749`
  (`/signal`), `crates/wiab-inf/src/sfu.rs:965-1297` (`handle_signal_socket`),
  join gate at `sfu.rs:1024-1034` →
  `crates/wiab-app/src/meeting_application_service.rs:85-98` (`validate_join`);
  `list_meetings` (`http_api.rs:169-179`) is itself unauthenticated.
- **Issue:** The `/signal` upgrade is a GET (so `csrf_guard` skips it) with no
  session check. The only gate on `JoinMeeting` is `validate_join`, which just
  requires the meeting to be active and the `participant_id` to be in the roster.
  `GET /meetings` returns the participant list including `participant_id`, so the
  needed identifier is handed out unauthenticated. Nothing binds the socket to an
  authenticated user.
- **Impact:** An attacker lists meetings, reads a valid `participant_id`, opens
  the WebSocket, joins the room, then consumes other participants' audio
  producers (`sfu.rs:1214`), injects audio, or ends the meeting for everyone
  (`EndMeeting`, `sfu.rs:1066`). Meeting hijacking and audio eavesdropping with no
  credential.
- **Recommendation:** Authenticate the WebSocket at upgrade (session cookie or
  token) and verify the authenticated user is the claimed participant before
  `create_peer`. Do not expose participant identifiers on an unauthenticated
  listing endpoint.

### H3 — Enterprise OIDC auto-links existing local accounts on an unverified email

- **Repo:** `backend`
- **Location:** `src/bootstrap.rs:361-364` (`auto_link_verified_email: true`,
  `require_email_verified: false` for the `enterprise` connection); linking logic
  `crates/authbox-app/src/federation_service.rs:127-146`.
- **Issue:** For the enterprise connection `require_email_verified` is false, so
  `email_verified` is never checked, yet `auto_link_verified_email` is true — so
  when `find_by_email(email)` matches an existing user, the SSO flow adopts that
  principal and mints a session for it, trusting an unverified email. The Google
  connection is correctly hardened (`auto_link_verified_email: false`,
  `require_email_verified: true`), which highlights the gap. The flag name is
  also misleading: it links *unverified* emails.
- **Impact:** If the enterprise IdP can be induced to emit a victim's email/UPN
  (a multi-tenant OIDC app, an IdP allowing self-asserted `email`/`preferred_
  username`, or a second provider sharing the domain), an attacker takes over the
  matching local account — including a password-based Owner — via SSO.
- **Recommendation:** Do not auto-link to a pre-existing local account on an
  unverified email. Require `email_verified`, or require an authenticated link
  step (log in with the existing credential, then attach the SSO identity). At
  minimum restrict enterprise auto-link to the IdP's verified domain(s).

### H4 — OIDC federation lets deactivated users log back in

- **Repo:** `backend`
- **Location:** `crates/authbox-app/src/federation_service.rs:112-114`; session
  established at `http_api.rs:1189-1193`.
- **Issue:** When an existing federated identity is found, the flow returns the
  linked principal with no active/enabled check. The password path is gated —
  `WiabUserDirectory::find_by_email` filters `user.is_active()`
  (`user_application_service.rs:71`) — but the existing-link SSO path bypasses
  `find_by_email` entirely.
- **Impact:** A user who was deactivated (which revokes sessions and bars
  password login, `http_api.rs:1235-1252`) but had previously linked SSO can
  re-authenticate through the IdP and obtain a fresh, fully authorized session.
  Deactivation is not an effective off-boarding control for SSO users.
- **Recommendation:** After resolving the principal in `complete_login` (both the
  existing-link and email-match branches), verify the host user is active before
  establishing a session; reject otherwise.

### H5 — TLS verification to Xen Orchestra is disabled

- **Repo:** `iac`
- **Location:** `terraform.tfvars:6` (`xoa_insecure = "true"`),
  `providers.tf:3-7`, `variables.tf:15-19`.
- **Issue:** The `xenorchestra` provider is configured to skip TLS verification
  to the XO websocket API (the hypervisor control plane), and the live tfvars
  sets it to `"true"`.
- **Impact:** The `xoa_token` (pool-admin-capable) is sent over a connection that
  accepts any certificate. An attacker on the path (ARP/DNS spoof on the LAN,
  rogue gateway) can MITM the session, steal the token, and take control of the
  entire XCP-ng pool — create/destroy/boot VMs, read disks.
- **Recommendation:** Issue XO a real certificate (internal CA or Let's Encrypt)
  and set `xoa_insecure = "false"`. If using a private CA, distribute its root to
  the Terraform host rather than disabling verification.

### H6 — `/etc/wiab/wiab.env` written world-readable with multiple plaintext secrets

- **Repo:** `iac`
- **Location:** `scripts/provision.sh:195-201` (`cat > /etc/wiab/wiab.env`, no
  `chmod`), appended by `main.tf:252-266` (`RESEND_API_KEY`,
  `WIAB_GOOGLE_CLIENT_SECRET`, `WIAB_OIDC_CLIENT_SECRET`) and `main.tf:162-164`
  (`DATABASE_URL` with the DB password).
- **Issue:** The file is created under root's default umask (mode 0644) and later
  secret lines are appended with `tee -a`, preserving 0644. It ends up holding the
  DB password, the Resend key, and the Google/OIDC client secrets, readable by
  every local account. (By contrast `provision.env` is correctly `0640` per
  `templates/cloud-init.yaml.tftpl:17`.)
- **Impact:** Any unprivileged local user or compromised non-root service on the
  host reads all application/identity secrets. As the systemd `EnvironmentFile`
  it must exist, but it should not be world-readable.
- **Recommendation:** `install -m 0640 -o root -g wiab /dev/null
  /etc/wiab/wiab.env` before writing, or `chmod 0640` (root:wiab) immediately
  after creation, and verify `tee -a` targets keep 0640.

### H7 — Untrusted microVM sandboxes are NAT'd onto the LAN with an accept-all forward policy

- **Repo:** `iac`
- **Location:** `scripts/provision.sh:104-123`
  (`DEFAULT_FORWARD_POLICY="ACCEPT"`, MASQUERADE of `${WIAB_MICROVM_SUBNET}` out
  the primary interface), `variables.tf:296-300` (`microvm_subnet` default
  `172.16.0.0/24`).
- **Issue:** IP forwarding is enabled and the microVM subnet is masqueraded out
  the host's default-route interface, with ufw's forward policy flipped to
  `ACCEPT` and no FORWARD filtering. Firecracker guests run agent-controlled code
  (the whole point of the sandbox) yet can reach the host's own services, the
  rest of the LAN, and the internet unrestricted.
- **Impact:** A malicious or prompt-injected agent inside a sandbox VM can pivot
  to the host's management surfaces (SSH, Postgres, the backend API), scan/attack
  other LAN hosts, or exfiltrate — undermining the isolation the microVMs exist to
  provide (SSRF / lateral-movement).
- **Recommendation:** Keep `DEFAULT_FORWARD_POLICY` at `DROP` and add explicit
  FORWARD rules: allow the microVM subnet outbound to the internet only; DROP
  guest→host and guest→RFC1918 (e.g. block `172.16.0.0/24 → 192.168.0.0/16`,
  `→ 10.0.0.0/8`, `→ host_ip`).

### H8 — Android release builds are signed with the well-known debug key

- **Repo:** `app`
- **Location:** `android/app/build.gradle:100-103`
  (`release { signingConfig signingConfigs.debug }`), signing block `:88-95`;
  `.github/workflows/release.yml:48-52` regenerates the debug keystore
  (`-storepass android -keypass android`) and publishes `app-release.apk` to
  GitHub Releases.
- **Issue:** The `release` build type reuses `signingConfigs.debug` — the AOSP
  debug identity (`CN=Android Debug`) with the universally known password
  `android`. CI regenerates that keystore per run and publishes the resulting APK
  publicly. (Note: the on-disk `debug.keystore` is correctly gitignored and never
  committed — the standard debug key, not a leaked release key. The real issue is
  using it for release.)
- **Impact:** A debug-signed APK provides no authenticity/integrity guarantee —
  anyone can build and sign an identical "Android Debug" app, so trojaned builds
  are indistinguishable from official ones and any future signature-based security
  (signature permissions, Play App Signing, App Links, server-side cert pinning)
  is defeated. A fresh key per CI run also breaks upgrade compatibility, and Play
  rejects debug-signed uploads.
- **Recommendation:** Create a real release keystore, store it as an encrypted CI
  secret (base64), add a dedicated `release` `signingConfig`, and stop reusing
  `signingConfigs.debug`. Do not publicly distribute debug-signed artifacts.

### H9 — Android app allows blanket cleartext HTTP and ws:// signaling, no TLS or pinning

- **Repo:** `app`
- **Location:** `android/app/src/main/AndroidManifest.xml:15`
  (`android:usesCleartextTraffic="true"`); `src/backendConfig.ts:5-14`
  (hardcoded `http://`, `SIGNAL_URL` derived as `ws://`).
- **Issue:** The production manifest enables cleartext traffic app-wide with no
  `networkSecurityConfig` scoping it to dev hosts. Backend URLs are `http://` and
  signaling is `ws://`. There is no certificate/public-key pinning anywhere.
- **Impact:** All REST and WebSocket signaling (SDP/DTLS/ICE negotiation, meeting
  control) can be transmitted in plaintext and intercepted or modified by a
  network attacker, including hijacking WebRTC transport setup. The blanket
  cleartext flag keeps HTTP downgrade possible even after a host switches to
  HTTPS. (Currently the shipped defaults point at loopback/emulator hosts, so no
  real endpoint is exposed yet — but the posture is production-unsafe the moment a
  LAN/remote host is set.) iOS ATS is correctly strict by contrast.
- **Recommendation:** Remove `usesCleartextTraffic="true"` from the main manifest
  (keep it only in `src/debug` if needed), serve backend over HTTPS/`wss://`, add
  a network security config restricting cleartext to explicit dev hostnames, and
  consider certificate pinning for the API/signaling host.

## Medium findings

### M1 — No rate limiting or lockout on authentication surfaces; Argon2 CPU-exhaustion DoS

- **Repo:** `backend`
- **Location:** `crates/wiab-inf/src/http_api.rs:962-996` (`login`), `:1069-1079`
  (reset request), `:1088-1109` (reset confirm), `:1280` (`signup`), `:1502-1519`
  (`issue_token`); git Basic auth `git_http.rs:219-255`; SSH `git_ssh.rs:92-106`.
  No limiter layer exists (grep for `RateLimit`/`governor`/`DefaultBodyLimit`
  returns nothing).
- **Issue:** No throttling, account lockout, or IP-based limiting on password
  login, reset-token confirmation, PAT presentation, or git auth. `login` is
  unauthenticated and CSRF-exempt and runs a 19 MiB Argon2id verify per attempt,
  with no maximum password length. Bodies are bounded only by axum's default.
- **Impact:** Unbounded online password brute-force / credential stuffing;
  brute-force of reset and access tokens; and concurrent large-password logins
  each allocating ~19 MiB in the blocking pool — a cheap anonymous memory/CPU
  exhaustion DoS.
- **Recommendation:** Add per-IP and per-account rate limiting with progressive
  backoff/lockout on login/reset/signup/token endpoints, cap password length
  (e.g. 128) before hashing, bound request body size, and cap concurrent password
  verifications.

### M2 — Username/email enumeration via login and signup response timing

- **Repo:** `backend`
- **Location:** `crates/authbox-app/src/authentication_service.rs:87-105` (early
  returns at `:92-97` before `verify`, acknowledged in the doc comment at
  `:83-86`); signup path `http_api.rs:1296-1313` →
  `crates/wiab-app/src/user_application_service.rs:90-98`.
- **Issue:** `login_with_password` returns `InvalidCredentials` immediately for an
  unknown email, running Argon2 only when a credential exists — a timing oracle.
  Signup returns 202 for both new and taken emails, but only the new-email path
  runs Argon2 `set_password` + email send, so the two are timing-distinguishable.
- **Impact:** An attacker distinguishes registered vs unregistered emails despite
  uniform status/error responses, then focuses brute-force/phishing. Combined with
  M1 (no rate limiting) this is practical.
- **Recommendation:** Run a dummy Argon2 verify against a constant PHC when no
  credential is found (flatten login timing), and perform equivalent work on both
  signup branches.

### M3 — Open-redirect bypass in `sanitize_return_to` via backslash

- **Repo:** `backend`
- **Location:** `crates/wiab-inf/src/http_api.rs:1124-1129`, used at `oidc_start`
  (`:1148-1150`) and the callback redirect (`:1204`).
- **Issue:** The guard accepts `next` when
  `value.starts_with('/') && !value.starts_with("//")`. A value like `/\evil.com`
  passes (second char is `\`, not `/`), but browsers normalize backslashes to
  forward slashes, so `Location: /\evil.com` resolves as protocol-relative
  `//evil.com`. The unsanitized value is stored in the auth flow and emitted
  verbatim by `Redirect::to(&return_to)` on the callback. (The server-side
  sanitizer does correctly block the plain `//` case — this is the residual
  bypass.)
- **Impact:** Open redirect on the SSO login flow — a trusted-looking app URL that
  lands the victim on an attacker site after a real login, useful for phishing.
- **Recommendation:** Reject any `next` containing `\`, or parse it and require a
  same-origin relative path (leading `/`, no scheme/authority, no second leading
  `/` or `\`). Re-sanitize on the callback as defence-in-depth.

### M4 — Path traversal in Firecracker role-image resolution via the agent template

- **Repo:** `backend`
- **Location:** `crates/wiab-inf/src/firecracker_runtime.rs:214-222`; validation
  gap `crates/wiab-core/src/vm/vm_template.rs:14-20` (`VmTemplate::new` accepts
  any non-empty string); reachable via the (unauthenticated) `create_agent` /
  `update_agent` `vm_type` (`http_api.rs:325`, `:374`).
- **Issue:** `images_dir.join(format!("{}.ext4", spec.template))` builds the
  read-only rootfs path from the template name with no allow-list, and
  `VmTemplate` performs no character validation. A `vm_type` such as
  `../../../../var/lib/wiab/images/../../some/secret` (resolving to a `*.ext4`
  file) escapes `images_dir` and is mounted as the guest's root filesystem.
- **Impact:** On the KVM host an attacker (unauthenticated, per C1) can mount an
  arbitrary host `*.ext4` file read-only inside a guest they control, disclosing
  its contents. A real host↔guest trust-boundary crossing, constrained to `.ext4`
  files.
- **Recommendation:** Resolve templates against a fixed allow-list
  (`base`, `developer`, …) or validate `VmTemplate` to `^[a-z0-9_-]+$`, and
  canonicalize + assert the resolved path stays within `images_dir`.

### M5 — Internal error detail and subprocess stderr leaked to clients

- **Repo:** `backend`
- **Location:** `crates/wiab-inf/src/http_api.rs:887-889` (`internal` returns
  `err.to_string()` with 500), `:1739-1740` (`bad_request`);
  `crates/wiab-inf/src/git_http.rs:300-306` (`spawn_failure` returns raw git
  stderr).
- **Issue:** 500/400 responses serialize the underlying `anyhow`/`git2`/DB error
  string straight to the client, and git spawn failures return
  `git <verb> failed: <stderr>`.
- **Impact:** Leaks internal filesystem paths, backend/DB error messages, and git
  internals — aiding reconnaissance (repo existence, on-disk layout, library
  versions).
- **Recommendation:** Return generic messages for 5xx with a correlation id and
  log the detail server-side; do not forward subprocess stderr to clients.

### M6 — Unbounded gzip decompression of git request bodies (decompression bomb)

- **Repo:** `backend`
- **Location:** `crates/wiab-inf/src/git_http.rs:199-214` (`decode_body`).
- **Issue:** With `Content-Encoding: gzip` the body is inflated via
  `GzDecoder::read_to_end(&mut out)` with no cap on output size. axum's default
  2 MB input limit bounds the compressed input but not the decompressed output, so
  amplification still applies.
- **Impact:** A small, highly compressible payload expands to arbitrary memory —
  a memory-exhaustion DoS.
- **Recommendation:** Decompress through a size-limited reader (`take(max_bytes)`)
  and reject bodies exceeding a sane bound; also set an explicit body limit on the
  git RPC routes.

### M7 — No HTTP security response headers (console, marketing site, and API)

- **Repos:** `frontend`, `website`, `backend`
- **Location:** `frontend/nginx.conf:1-24` (no `add_header` at all);
  `website/firebase.json:11-30` (only `Cache-Control`); backend responses set no
  security headers (`http_api.rs`).
- **Issue:** None of `Content-Security-Policy`, `X-Frame-Options` /
  `frame-ancestors`, `X-Content-Type-Options: nosniff`, `Referrer-Policy`,
  `Permissions-Policy`, or `Strict-Transport-Security` are emitted. The console
  (`frontend`) is an admin surface managing users, tokens, and SSH keys; the
  website loads a third-party GA script and remote fonts with no CSP.
- **Impact:** No clickjacking protection on the admin console (UI-redress against
  destructive actions), no CSP defence-in-depth against any future injection or a
  compromised GA tag, and MIME-sniffing/referrer leakage. (Firebase Hosting does
  emit platform HSTS for the website; the app-level headers are the gap.)
- **Recommendation:** Add a hardened header block on the nginx `**` location and
  a `firebase.json` `headers` entry: `X-Frame-Options: DENY` (or CSP
  `frame-ancestors 'none'`), `nosniff`, `Referrer-Policy:
  strict-origin-when-cross-origin`, a scoped `Permissions-Policy`, and a CSP
  scoped to the actual origins used (self + `googletagmanager.com` /
  `google-analytics.com` / Google Fonts for the website; `default-src 'self'` for
  the console). Test with `Content-Security-Policy-Report-Only` first.

### M8 — Website pre-launch gate is client-side only; unreleased content ships to everyone

- **Repo:** `website`
- **Location:** `src/components/LaunchGate.tsx:14-21`, `src/lib/launch.ts:10-22`,
  `src/App.tsx:5-10`.
- **Issue:** `LaunchGate` only chooses which React subtree renders; all gated
  pages are statically imported in `App.tsx` (no `React.lazy`/`import()`
  anywhere), so 100% of the unreleased content is compiled into the main bundle
  and served to every visitor regardless of `VITE_LAUNCHED` or the email
  allowlist. `isPreviewer()` is a client-side `Array.includes` anyone can bypass.
- **Impact:** The confidentiality the gate appears to provide is illusory — any
  visitor can read all pre-launch copy, pricing, and product messaging from the
  bundle. The source comment correctly acknowledges this; flagged so the pattern
  is never reused for anything sensitive.
- **Recommendation:** Treat the gate as UX only. If content must truly stay
  private, use a Firebase Hosting preview channel behind an expiring URL (the repo
  already uses these), edge Basic auth, or don't build the unreleased pages into
  the public bundle. For marketing copy, accepting the exposure is defensible —
  document that decision.

### M9 — Weak default PostgreSQL password (`wiab`)

- **Repo:** `iac`
- **Location:** `variables.tf:162-167` (default `"wiab"`), `terraform.tfvars:49`
  (live value is the default), used in `scripts/provision.sh:185,199` and
  `main.tf:159,164`.
- **Issue:** The `wiab` Postgres role uses the trivial password `wiab`, kept in
  the live tfvars, and it appears in `DATABASE_URL` in the world-readable
  `wiab.env` (H6).
- **Impact:** Postgres binds localhost, limiting remote risk — but combined with
  the microVM egress gap (H7) and the world-readable `DATABASE_URL`, a
  low-privilege local process or a sandbox that reaches the host can trivially
  authenticate to the DB.
- **Recommendation:** Generate a strong random password (`random_password`), never
  ship a memorable default, and rotate the current value (it is in state/tfvars).

### M10 — SSH exposed to any source; password/root login not explicitly hardened

- **Repo:** `iac`
- **Location:** `scripts/provision.sh:298` (`ufw allow OpenSSH`, any source),
  `templates/cloud-init.yaml.tftpl:6-13`; no `PasswordAuthentication` /
  `PermitRootLogin` / `ssh_pwauth` set anywhere.
- **Issue:** ufw opens port 22 to all sources and cloud-init relies entirely on
  the base image's sshd defaults — it never explicitly disables password auth or
  root login. The example `announced_address` is a public IP, signalling the host
  may sit behind WAN NAT (SSH reachable from the internet). The `ubuntu` user also
  has `NOPASSWD:ALL` sudo.
- **Impact:** If NAT-exposed, port 22 is open to the internet with hardening left
  to image defaults (brute-force surface).
- **Recommendation:** Restrict SSH to trusted source CIDRs
  (`ufw allow from <admin-cidr> to any port 22`) and explicitly assert
  `PasswordAuthentication no` + `PermitRootLogin no` via a cloud-init drop-in.

### M11 — Secrets rendered into cloud-init user-data (readable via the hypervisor and state)

- **Repo:** `iac`
- **Location:** `main.tf:36-60` + `templates/cloud-init.yaml.tftpl:28-35`
  (`WIAB_DB_PASSWORD`, the `WIAB_MODELS_URL`/`WIAB_IMAGES_URL` SAS tokens rendered
  into `provision.env`).
- **Issue:** The DB password and Azure SAS URLs are templated into the VM's
  cloud-init, which is retrievable by anyone with XO/pool access and is stored
  rendered in `terraform.tfstate`.
- **Impact:** A pool operator with XO read access — or anyone with the state file
  (C3) — recovers the DB password and long-lived SAS tokens without touching the
  guest, broadening the blast radius of C3 and H5.
- **Recommendation:** Avoid durable secrets in user-data; fetch them on the guest
  from a secrets manager at first boot, or inject via the SSH `remote-exec` path.
  Scope SAS tokens tightly (read-only, short TTL, single container).

### M12 — Downloaded binaries and kernels are not checksum- or signature-verified

- **Repo:** `iac`
- **Location:** `scripts/provision.sh:45-48` (firecracker/jailer tarball),
  `:63-64` (smoke-test `vmlinux` + `rootfs.ext4`), `scripts/wiab-deploy.sh:93-95`
  (azcopy from `aka.ms`), `images/kernel/fetch.sh:34`, `images/kernel/build.sh:
  123-124` (kernel source), `.github/workflows/images.yml:70-72` (azcopy).
- **Issue:** Firecracker, jailer, azcopy, the CI kernel/rootfs, and the guest
  kernel source are pulled over HTTPS with no pinned SHA-256 or signature check.
  Transport is HTTPS (integrity-of-artifact, not cleartext), but a compromised or
  rotated upstream artifact — or a TLS-terminating proxy — is not detected.
- **Impact:** A tampered firecracker/jailer or guest kernel executes as the VMM /
  PID 1 across the sandbox fleet — a high-value supply-chain target.
- **Recommendation:** Pin and verify SHA-256 (Firecracker publishes per-release
  checksums; azcopy and kernel tarballs can be pinned to a digest). The
  backend/frontend deploy path already does this (`wiab-deploy.sh:275-280,
  332-337`) — extend the same discipline to the VMM, kernel, and azcopy.

### M13 — Third-party GitHub Actions pinned by tag, not commit SHA

- **Repos:** all with CI (`website`, `app`, `docs`, `iac`, `backend`, `frontend`,
  `dev`)
- **Location:** every `*/.github/workflows/*.yml`. Notable non-GitHub actions:
  `FirebaseExtended/action-hosting-deploy@v0` (website), `softprops/action-gh-
  release@v2` (app), `DavidAnson/markdownlint-cli2-action@v20` (docs),
  `hashicorp/setup-terraform@v3` (iac), `gradle/actions/setup-gradle@v4` (app).
- **Issue:** Mutable tags (`@v0`, `@v2`, `@v4`) let the upstream owner — or an
  attacker who compromises the action repo — change the code a workflow runs with
  no change here. `FirebaseExtended/action-hosting-deploy@v0` is especially
  notable: a pre-1.0 float running inside a job that holds
  `FIREBASE_SERVICE_ACCOUNT`.
- **Impact:** A compromised third-party action executes with the job's
  `GITHUB_TOKEN` and any in-scope secrets (Firebase service account, image SAS,
  `contents: write`).
- **Recommendation:** Pin third-party actions to a full commit SHA with the
  version in a trailing comment; enable Dependabot for `github-actions`.
  GitHub-owned `actions/*` are lower risk but ideally SHA-pinned too.

### M14 — Privileged Firebase service account exposed to PR/tag builds that run repo code

- **Repo:** `website`
- **Location:** `.github/workflows/firebase-preview.yml` (`on: pull_request`,
  deploy with `secrets.FIREBASE_SERVICE_ACCOUNT` at line 44) and `release.yml`
  deploy job (line 159).
- **Issue:** `firebase-preview.yml` runs `npm ci` + `npm run build` (executing
  lockfile lifecycle scripts and Vite plugins from the branch) and then deploys
  with the Firebase service account, all in one runner. Forks are correctly gated
  off (`head.repo.full_name == github.repository`), so this affects same-repo
  branches only — but any collaborator/compromised branch runs arbitrary build
  code alongside a deploy-capable GCP credential.
- **Impact:** A malicious build step or poisoned npm lifecycle script could
  exfiltrate the service account or tamper with what is deployed.
- **Recommendation:** Split "build untrusted code" from "deploy with secret" into
  separate jobs so the service account is never present while PR code runs; prefer
  OIDC/workload-identity over a long-lived JSON; use `npm ci --ignore-scripts`
  where feasible.

### M15 — Mobile app has no client authentication; ownership enforced only in the UI

- **Repo:** `app`
- **Location:** `App.tsx:306` (`fetch(.../meetings)`, no auth header), `:646-650`
  (`join_meeting`), `:719-733` (`end_meeting`), `:735-738` + `:774-781`
  (`isOwner` gates the "End Meeting" button in the UI only).
- **Issue:** The app sends no credentials/tokens on any request. Listing, joining,
  producing/consuming audio, and ending meetings use unauthenticated, guessable
  identifiers, and the owner check gating the destructive action is purely a
  client-side UI condition.
- **Impact:** Anyone who can reach the backend can enumerate/join meetings, stream
  audio, and (by crafting `end_meeting` directly) terminate meetings they don't
  own — unless the server independently enforces identity and authorization (it
  does not; see H2). From the client's perspective there is zero access control.
- **Recommendation:** Introduce authenticated sessions and enforce all
  authorization server-side (pairs with H2). Treat the client `isOwner` gate as
  cosmetic.

### M16 — Frontend dependency vulnerabilities (dev-server and transitive)

- **Repo:** `frontend`
- **Location:** `package.json` — `vitest ^3.0.0` (resolved `<3.2.6`),
  `vite 7.1.1`, `axios ^1.17.0` (transitive `form-data 4.0.0–4.0.5`).
- **Issue:** `npm audit` reports 1 critical / 2 high / 1 moderate. GHSA-5xrq-8626-
  4rwp (vitest UI arbitrary file read/exec, dev-only), a cluster of Vite
  dev-server `server.fs` bypass / arbitrary-file-read advisories (build-time
  only), and GHSA-hmw2-7cc7-3qxx (`form-data` CRLF injection) pulled via the
  production `axios` dependency.
- **Impact:** The vitest and vite issues affect only the local dev/preview server
  (never the production static bundle) and require the dev server to be reachable,
  so real-world risk is limited. `form-data` sits in the production dependency
  tree, though the browser build uses native `FormData` and generally does not
  bundle the vulnerable path. `website` already pins a clean Vite (`^7.3.5`);
  `frontend` is pinned to an exact old version.
- **Recommendation:** `npm audit fix` (bumps vitest ≥3.2.6 and `form-data`
  ≥4.0.6), bump Vite to ≥7.3.6 (align with `website`), wire `npm audit` into CI,
  and never expose the Vite dev server on untrusted networks.

### M17 — Android release APK is unminified/unobfuscated and published publicly

- **Repo:** `app`
- **Location:** `android/app/build.gradle:60`
  (`enableProguardInReleaseBuilds = false`), applied at `:104`; a release JS
  source map is produced; `.github/workflows/release.yml` publishes
  `app-release.apk`.
- **Issue:** R8/ProGuard is disabled for release, so the JS bundle and native code
  ship without minification/obfuscation and a release source map is generated; the
  unobfuscated APK is attached to public GitHub Releases.
- **Impact:** Trivial reverse engineering of app logic and internal
  signal/protocol structure. (No secrets are embedded, limiting the blast radius.)
- **Recommendation:** Enable `minifyEnabled true` + R8 for release, keep JS source
  maps out of distributed artifacts, and reconsider publishing raw APKs publicly.

## Low findings

### L1 — Ephemeral Git SSH host key when unconfigured

- **Repo:** `backend`
- **Location:** `crates/wiab-inf/src/git_ssh.rs:41-44`; bind `src/main.rs:79`
  (`0.0.0.0:2222`).
- **Issue:** If `WIAB_GIT_SSH_HOST_KEY` is unset, a new host key is generated on
  every start, so clients cannot meaningfully pin it.
- **Impact:** Changing host keys trains users to accept new keys, undermining
  SSH's MITM protection on a `0.0.0.0`-exposed port.
- **Recommendation:** Require a persistent host key in any non-dev deployment
  (fail to start rather than generating an ephemeral one when the base URL /
  `cookie_secure` indicate production).

### L2 — Agent vsock reports logged unsanitized and unbounded

- **Repo:** `backend`
- **Location:** `crates/wiab-inf/src/vm_comms_broker.rs:42-46`.
- **Issue:** Lines read from the per-VM Unix socket (written by untrusted in-guest
  agent code) are logged verbatim via `tracing::info!`, and `next_line` buffers
  an unbounded line. (Good: the broker only logs, takes no action on content, and
  each socket path is per-VM so cross-VM access is not possible.)
- **Impact:** Log forging (newline/control-char injection) and unbounded line
  buffering from a compromised agent. Not a privilege-boundary break.
- **Recommendation:** Truncate/escape control characters and cap line length
  before logging; treat all guest input as untrusted.

### L3 — Token/key authentication loads every user on each request

- **Repo:** `backend`
- **Location:** `crates/wiab-app/src/user_application_service.rs:261-312`
  (`resolve_token` / `resolve_user_by_fingerprint`);
  `crates/wiab-inf/src/postgres_user_repository.rs:238-257` (`list` loads all
  users + tokens).
- **Issue:** Every Bearer/Basic/SSH-key request calls `user_repository.list()` and
  scans all users in memory to match the token hash / key fingerprint.
- **Impact:** O(N-users) DB work per authenticated request — a scaling and
  amplification-DoS concern as the user table grows.
- **Recommendation:** Look up tokens/keys by an indexed hash/fingerprint column in
  SQL (`WHERE hash = $1`) instead of listing all users.

### L4 — Dev email sender logs the full single-use reset/invite link

- **Repo:** `backend`
- **Location:** `crates/authbox-inf/src/logging_email_sender.rs:9-12`.
- **Issue:** `LoggingEmailSender` logs recipient, subject, and full body — which
  includes the reset/invite/verify URL with the plaintext token — at INFO. It is
  the fallback when no mailer is configured.
- **Impact:** If ever selected in a shared/prod environment, single-use tokens
  leak to logs, enabling account takeover within the token TTL.
- **Recommendation:** Redact the token/link (log recipient + purpose only), or
  gate this sender to a debug build / explicit dev flag and ensure production
  always uses Resend/SMTP.

### L5 — Secret-bearing structs derive `Debug` with plaintext secrets

- **Repo:** `backend`
- **Location:** `crates/authbox-app/src/authentication_service.rs:18-22`
  (`EstablishedSession { cookie_secret, csrf_token }`);
  `crates/authbox-core/src/auth/oidc_port.rs:6-11`
  (`AuthRequest { state, nonce, pkce_verifier }`).
- **Issue:** These hold plaintext session/CSRF secrets and PKCE/nonce and derive
  `Debug`. Nothing currently logs them (verified), so this is latent.
- **Impact:** A future `debug!`/`{:?}`/error-context on these values would leak
  live session or in-flight OIDC secrets to logs.
- **Recommendation:** Implement a redacting `Debug` or wrap secrets in a
  `Secret`/`Redacted` newtype.

### L6 — Weak and inconsistently enforced password policy

- **Repo:** `backend`
- **Location:** `crates/wiab-inf/src/http_api.rs:1039,1092,1290,1328` (all
  `len() < 8`); no check in `AuthenticationService::set_password`
  (`authentication_service.rs:179`).
- **Issue:** The only rule is an 8-character minimum, enforced ad hoc in HTTP
  handlers. The core `set_password`/`change_password` services accept any string
  (including empty), so any path that skips the handler check bypasses the policy.
- **Impact:** Weak passwords accepted; policy bypassable by any non-handler
  caller.
- **Recommendation:** Centralize a password policy in the app service (length
  floor + ceiling, optional breached-password check).

### L7 — CSRF token comparison is not constant-time

- **Repo:** `backend`
- **Location:** `crates/authbox-app/src/authentication_service.rs:163-165`
  (`csrf_matches`).
- **Issue:** The presented token is hashed and compared with `==` against the
  stored CSRF hash. This compares hash outputs (not the secret directly), so an
  attacker cannot practically forge via a prefix-timing oracle, but it is not
  constant-time.
- **Impact:** Minimal — comparing high-entropy hash outputs with `==` is not a
  realistic forgery vector; noted for completeness/hardening.
- **Recommendation:** Use a constant-time comparison (`subtle`/`ct_eq`) for the
  CSRF hash comparison to match the rest of the crypto discipline.

### L8 — nginx disables upstream TLS verification to the backend

- **Repos:** `frontend`, `iac`
- **Location:** `frontend/nginx.conf:18` (`proxy_ssl_verify off`);
  `iac/scripts/provision.sh:258-260`, `iac/main.tf:199-202`.
- **Issue:** The `/api/` reverse proxy to the backend sets `proxy_ssl_verify off`
  (accommodating the backend's self-signed cert), so nginx does not authenticate
  the backend certificate.
- **Impact:** On the deployed host this is a loopback hop (acceptable), but in the
  container topology any attacker able to interpose on the nginx→backend hop could
  MITM authenticated cookies/CSRF tokens undetected.
- **Recommendation:** Pin the backend CA
  (`proxy_ssl_trusted_certificate` + `proxy_ssl_verify on` + `proxy_ssl_name
  backend`) rather than disabling verification, even for internal self-signed
  certs.

### L9 — Frontend Docker uses floating base tags and runs nginx as root

- **Repo:** `frontend`
- **Location:** `Dockerfile:4` (`FROM node:22-bookworm`), `:19`
  (`FROM nginx:alpine`).
- **Issue:** Base images use mutable tags rather than digest pins (supply-chain
  drift), and the final `nginx:alpine` stage sets no `USER`, so nginx runs as
  root.
- **Impact:** A poisoned upstream tag could alter the shipped image with no repo
  change; running as root widens the blast radius of any nginx/container escape.
- **Recommendation:** Pin base images by digest and bump deliberately; use
  `nginxinc/nginx-unprivileged` or add a non-root `USER`.

### L10 — nginx version disclosure and unconditional WebSocket upgrade forwarding

- **Repo:** `frontend`
- **Location:** `nginx.conf` (`server_tokens` unset → `on`), `:20-22`
  (Upgrade/Connection headers).
- **Issue:** `server_tokens` is not disabled (leaks the exact nginx version), and
  the `/api/` location forwards client-controlled `Upgrade`/`Connection: upgrade`
  headers on every request, not just WebSockets.
- **Impact:** Version disclosure aids targeted exploitation (informational);
  unconditional upgrade forwarding is a minor request-smuggling/oddity surface.
- **Recommendation:** Set `server_tokens off;` and gate the `Connection` header via
  a `map $http_upgrade $connection_upgrade { default upgrade; '' close; }`.

### L11 — Previewer email allowlist baked into the public website bundle

- **Repo:** `website`
- **Location:** `src/lib/launch.ts:13-18` (fed by `VITE_PREVIEW_ALLOWLIST`,
  injected at build in `release.yml`).
- **Issue:** Because it is a `VITE_*` var, the allowlist is inlined into the
  shipped JavaScript; the plaintext emails of privileged previewers are readable
  by anyone who opens the bundle, despite being stored as a GitHub Secret.
- **Impact:** Discloses insider/staff emails (minor PII) useful for targeted
  phishing against the exact accounts that can sign in pre-launch.
- **Recommendation:** Move the allowlist check server-side (Firebase custom claims
  or a Cloud Function) so client code only checks an authenticated token; at
  minimum stop treating the list as secret.

### L12 — Website cookie consent cannot be withdrawn after acceptance

- **Repo:** `website`
- **Location:** `src/components/CookieBanner.tsx:12-14`,
  `src/analytics/analytics.ts:19-26`.
- **Issue:** The banner shows only while consent is `null`; once accepted there is
  no UI to revisit or withdraw, and GA stays loaded. For an EU/Danish operator,
  GDPR/ePrivacy require withdrawal to be as easy as granting.
- **Impact:** Compliance gap; an accepting user has no in-product way to opt back
  out.
- **Recommendation:** Add a persistent "Cookie settings" link (e.g. in the footer)
  that reopens the banner and can set consent to `denied`; clear GA cookies on
  withdrawal. The opt-in model itself is otherwise correct.

### L13 — Release workflows grant `contents: write` at workflow scope

- **Repos:** `backend`, `frontend`, `website`, `dev`, `app`
- **Location:** top-level `permissions: contents: write` in each `release.yml`.
- **Issue:** The write token is granted to every job (build/test/package), not
  just the job that creates the release.
- **Impact:** Any compromised step in a build job (see M13) can push
  commits/tags/releases.
- **Recommendation:** Set the workflow default to `contents: read` and grant
  `contents: write` only on the create-release/upload/deploy job.

### L14 — Tag-derived values interpolated into `run:` shell (latent injection)

- **Repos:** `backend`, `frontend`, `website`, `dev`
- **Location:** `run:` blocks using `${{ steps.get_tag.outputs.tag }}` /
  `${{ needs.create-release.outputs.version }}` — e.g. `backend/release.yml:
  37-38,152-153`.
- **Issue:** These derive from the pushed git tag (`GITHUB_REF`); interpolating
  them into bash means a crafted tag could inject shell. Requires push access, so
  the trust boundary is maintainer-only.
- **Impact:** Low (privileged trigger), but a real `${{ }}`-into-`run:` sink.
- **Recommendation:** Pass through `env:` and reference as `"$TAG"`/`"$VERSION"`
  (the env-indirection already used correctly in `iac/images.yml:30-32`).

### L15 — Unpinned Docker service image in backend CI

- **Repo:** `backend`
- **Location:** `.github/workflows/ci.yml:95` (`axllent/mailpit`, no tag →
  floating latest; `postgres:16` and the mock OIDC server are pinned).
- **Issue:** The mailpit test-service image is unpinned.
- **Impact:** Low — test-only, no secrets, but non-reproducible and a soft
  supply-chain surface.
- **Recommendation:** Pin `axllent/mailpit@sha256:…` or a version tag.

### L16 — backend/.gitignore has no env/secret patterns

- **Repo:** `backend`
- **Location:** `backend/.gitignore` (covers only build artifacts).
- **Issue:** Unlike the other repos it has no `.env`/secret ignore rules. Backend
  configures via process env/clap today (no `.env` tracked or present), so nothing
  is exposed — but there is no guardrail if a dev drops a `.env` with
  `DATABASE_URL`/SMTP/OIDC creds.
- **Impact:** Low / preventive.
- **Recommendation:** Add `.env`, `.env.*`, `!.env.example` to
  `backend/.gitignore`.

### L17 — iOS declares a location permission with an empty usage description

- **Repo:** `app`
- **Location:** `ios/workinabox/Info.plist:34-35`
  (`NSLocationWhenInUseUsageDescription` empty).
- **Issue:** A location purpose string is declared but blank; the app never
  requests location.
- **Impact:** App Store review may reject a declared permission with no purpose
  string; a dependency-triggered prompt would show a blank rationale. Signals a
  broader-than-needed permission surface.
- **Recommendation:** Remove the key if location is unneeded; otherwise provide a
  real description.

### L18 — Mobile backend URLs hardcoded cleartext with no HTTPS path

- **Repo:** `app`
- **Location:** `src/backendConfig.ts:5-14`.
- **Issue:** `HTTP_BASE_URL`/`SIGNAL_URL` are always `http://`/`ws://`; setting
  `LAN_BACKEND_HOST` yields `http://<host>:8080` with no TLS option, and the URL
  is surfaced in the UI.
- **Impact:** Forces plaintext transport for real-device testing on shared Wi-Fi;
  no code path points at HTTPS. (Related to H9.)
- **Recommendation:** Support `https://`/`wss://` and default to TLS for any
  non-loopback host.

### L19 — Secrets and map values interpolated into shell/SQL command strings

- **Repo:** `iac`
- **Location:** `main.tf:159`
  (`ALTER ROLE wiab LOGIN PASSWORD '${var.db_password}'`), `:164`
  (`DATABASE_URL=…${var.db_password}…`), `:253-266` (echoed secrets), `:320`;
  config sourced via `set -a; . /etc/wiab/provision.env`.
- **Issue:** Operator-supplied secrets and the `models` map are interpolated
  directly into `remote-exec` shell/SQL. A value with a quote, `$`, `;`, or `&`
  breaks the command or, in the sourced-env case, is interpreted by the shell;
  single-quoted `provision.env` values break on a literal `'`.
- **Impact:** Mostly robustness (inputs are operator-controlled), but a
  special-char secret (common in generated keys) can corrupt the DB password or
  the env file, or execute unintended shell.
- **Recommendation:** Write secrets via files/heredocs with quoting, use `psql`
  variable binding for the role password, prefer a non-shell-sourced env format,
  and constrain generated secrets to a safe charset.

### L20 — mediasoup binds 0.0.0.0 with a very wide UDP range open to any source

- **Repo:** `iac`
- **Location:** `scripts/provision.sh:196` (`WIAB_MEDIASOUP_LISTEN_IP=0.0.0.0`),
  `:301-303` (`ufw allow 10000:59999/udp`).
- **Issue:** mediasoup listens on all interfaces and ~50,000 UDP ports are opened
  to any source (a workaround for the backend using an unbounded RTC port range).
- **Impact:** Large exposed UDP surface; expected for public WebRTC media but
  broader than necessary and internet-open if NAT-forwarded.
- **Recommendation:** Pin a bounded RTC port range in the backend and narrow the
  ufw rule; restrict source where the topology allows.

## Informational

- **I1 — Self-signed backend TLS default and `0.0.0.0` bind (`backend`).**
  `src/main.rs:92,109-134`, SSH `:79`. With `WIAB_TLS_CERT`/`KEY` unset the
  backend serves a self-signed cert and the code comment invites proxies to skip
  verification; no security headers are set on responses. Fine behind a
  verifying/pinned proxy on loopback; require a real cert (or pinned internal CA)
  in production and keep verification on. (Related to L8, M7.)
- **I2 — Verbose IdP errors returned to the client (`backend`).**
  `federation_service.rs` surfaces detailed OIDC validation/discovery errors at
  `http_api.rs:1188`. Minor config info disclosure; return a generic "SSO login
  failed" and log detail server-side.
- **I3 — Firebase web API key is public by design (`website`).**
  `src/config/firebase.ts:4-11`. Expected for Firebase web apps (an identifier,
  not a secret); already documented. Restrict the browser key in GCP (HTTP-
  referrer allowlist) and keep Auth authorized domains tight.
- **I4 — No Firestore/Storage/RTDB rules files, but the client uses only Firebase
  Auth (`website`).** No `firestore`/`getStorage`/`database` usage anywhere.
  Acceptable today; confirm those services are disabled or default-deny for the
  project and add deny-by-default rules before any data feature ships.
- **I5 — i18n interpolation runs with `escapeValue: false` (`website`).**
  `src/i18n/index.ts:31`. The react-i18next default; React escapes text children
  and all translation values are static, so no XSS today. Never feed user input
  through `t()`/`Trans` with raw HTML.
- **I6 — Google Fonts loaded without Subresource Integrity and no CSP
  (`frontend`, `website`).** A third-party origin in the trust boundary (visitor
  IP leak; CDN-compromise CSS injection). Self-host the fonts or allow only those
  hosts in a CSP.
- **I7 — Native `AgentAudioPlayer` module plays arbitrary base64 audio, currently
  unused (`app`).** `android/.../AgentAudioPlayerModule.kt:21-25`,
  `ios/.../AgentAudioPlayer.m`. No JS references it. When wired in, treat server
  audio as untrusted (media-parser surface); keep using internal cache storage.
- **I8 — `cargo audit` not installed; manual Rust dependency scan clean
  (`backend`, `dev`).** No automated RUSTSEC scan was possible. A manual review of
  locked versions (axum 0.8.8, tokio 1.50.0, rustls 0.23.40, ring 0.17.14, hyper
  1.8.1, git2 0.20.4 / libgit2 1.9.4, argon2 0.5.3, russh 0.61.2, openidconnect
  4.0.1, …) found no known-vulnerable pins and no git-sourced dependencies. Wire a
  `rustsec/audit-check` CI job so advisories are caught going forward.
- **I9 — LICENSE files missing in `docs` and `assets`.** All other repos and the
  org `.github` repo carry MIT. Add for consistency.
- **I10 — No deep links, custom URL schemes, or WebViews in the mobile app
  (`app`).** `MainActivity` is `exported="true"` only for the standard
  `MAIN`/`LAUNCHER` filter (expected). The deep-link and WebView-bridge risk
  classes do not apply; re-audit if schemes/App Links or WebViews are added.

## Dependency and supply-chain audit

| Repo | Tool | Critical | High | Moderate | Low |
|------|------|:--------:|:----:|:--------:|:---:|
| `backend` | manual (cargo-audit absent) | 0 | 0 | 0 | 0 |
| `dev` | manual (cargo-audit absent) | 0 | 0 | 0 | 0 |
| `app` | `npm audit` | 0 | 0 | 1 | 0 |
| `frontend` | `npm audit` | 1 | 2 | 1 | 0 |
| `website` | `npm audit` | 0 | 0 | 0 | 1 |

Notes:

- `app`/`frontend`/`website` counts include devDependencies. The `website`
  production tree is clean; its one Low is dev tooling.
- The `frontend` critical/high are the vitest and Vite dev-server advisories plus
  the transitive `form-data` — all detailed in M16; the two dev-server issues do
  not affect the shipped static bundle.
- Rust rows are a manual pin review (I8), not an automated scan.
- No dependency is pulled from a git/URL source; no typosquat-looking packages; no
  `overrides`/`resolutions` forcing old versions; all lockfiles present.

## Secret and credential file inventory

Local files holding real secrets (checked for git-tracking; all correctly
untracked/ignored unless noted):

| File | Repo | Contains | Tracked in git? |
|------|------|----------|-----------------|
| `terraform.tfstate` / `.backup` | `iac` | XO token, DB pw, OIDC/Google secrets, Resend key, SAS URLs (rendered) | No (gitignored) — but plaintext on disk, see C3 |
| `terraform.tfvars` | `iac` | live values of all the above | No (gitignored) — see C3 |
| `dev/local/oidc.env` | `dev` | OIDC/Google client secrets, Resend key, dev owner password | No (gitignored) |
| `frontend/.env` | `frontend` | non-secret build flags only | No (gitignored) |
| `website/.env.local` | `website` | GA measurement ID only (non-secret) | No (gitignored) |
| `app/ios/.xcode.env` | `app` | stock React Native `NODE_BINARY` line (no secret) | Yes (benign, expected) |
| `app/android/app/debug.keystore` | `app` | standard AOSP debug key (password `android`) | No (gitignored) |

Git-history scan across all repos for committed secret **values**
(`re_…`, `AKIA…`, `AIza…`, `ghp_…`, `sig=…`, PEM private-key blocks) returned
nothing. The `client_secret`/`api_key`/`password` string hits in history are
config field and variable names, not credentials.

## Good practices observed

This codebase gets a large amount right, and the findings above should be read
against that baseline:

- **Password hashing:** Argon2id at the OWASP baseline (m = 19 MiB, t = 2, p = 1),
  a fresh 16-byte CSPRNG salt per hash, PHC-string storage, verification via the
  library's constant-time comparator, all on `spawn_blocking`.
- **Secret generation and storage:** session, CSRF, and verification/reset/invite
  secrets are 256-bit values from a CSPRNG (`rand::rng()`), stored only as SHA-256
  hashes; comparisons run on the hashes. SHA-256 (not a slow KDF) is the correct
  choice for these high-entropy tokens.
- **Sessions:** separate idle and absolute expiries, absolute cap never extended
  on touch, explicit revoke, logout invalidation, `revoke_all_for_principal` on
  password reset, and a unique index on `token_hash`.
- **CSRF:** double-submit token bound to the session hash, enforced only on
  cookie-authenticated unsafe methods (Bearer/Basic exempt), with `HttpOnly` +
  `SameSite=Lax` session cookie and `Secure` over HTTPS; no state-changing GET
  routes. The web console stores the session only in the HttpOnly cookie — never
  in `localStorage` — so it is not XSS-exfiltratable.
- **OIDC:** PKCE (S256), single-use `state`, nonce validation, and ID-token
  signature/iss/aud/exp validation delegated to the vetted `openidconnect` crate
  with JWKS; the outbound HTTP client disables redirect-following (SSRF
  hardening); identities key on the durable `subject`, not email; Google refuses
  silent account linking; `next` is sanitized server-side (modulo M3).
- **Injection resistance:** every Postgres access is parameterized (no string
  interpolation into SQL); git and VM subprocesses are spawned with explicit argv
  (never `sh -c`) and whitelisted verbs; `RepoId` is a strict `R-<u64>` and branch
  names are validated against `..`/control/glob metacharacters, so repo path
  resolution cannot traverse.
- **git transport:** anonymous clone/fetch only for `Public` repos, otherwise
  token + role; push always requires Write; SSH is public-key-only (password/none
  rejected, offered key must resolve to a user).
- **Account-existence non-disclosure:** signup and password-reset always return
  202; reset swallows mailer errors to avoid an oracle (timing aside, M2).
- **Sandbox:** microVMs run under jailer (chroot + unprivileged) with a tightly
  scoped `sudoers.d` allow-list and a VM-id-regex-validated helper; the Firecracker
  version is pinned (never `latest`) and gated by a real boot smoke test;
  read-only base rootfs + per-instance overlay; per-VM vsock sockets are
  path-isolated.
- **Deploy:** backend/frontend release artifacts are SHA-256 verified before
  install with health-checked deploy and automatic rollback; SSH keys for guest
  images come from a CI secret, never hardcoded; CI grants least-privilege
  `contents: read` on non-release workflows and closes fork-PR secret exposure.
- **Repo hygiene:** no secrets in any git history; `.gitignore`s cover
  env/state/keystore files; DB URL password is redacted before logging.

## Prioritized remediation roadmap

Ordered by risk-reduction per unit effort.

1. **Close the API authorization gap (C1, C2).** Add a global authentication
   middleware with an explicit public allow-list, then a per-resource
   `require_*` check on every handler, with repo reads gated by visibility/role.
   This single change removes both Criticals and the unauthenticated-access
   precondition behind H2, M4, and M15. Add integration tests asserting 401/403
   for every route without a credential.
2. **Remediate infrastructure secret exposure (C3, H6, M9, M11).** Move Terraform
   to an encrypted, locked remote backend; rotate every secret in the current
   state/tfvars (XO token, DB password, Resend key, Google + OIDC client secrets,
   both SAS tokens); make `wiab.env` `0640`; generate a strong DB password; stop
   rendering durable secrets into cloud-init.
3. **Fix the boot backdoor and credential logging (H1, L4, L5).** Refuse the
   default owner password in production, stop logging the password and bootstrap
   token, redact the logging email sender, and add redacting `Debug` to
   secret-bearing structs.
4. **Authenticate real-time media and harden SSO (H2, H3, H4, M3).** Auth the
   `/signal` WebSocket at upgrade and bind it to the participant; require
   `email_verified` (or an explicit link step) before enterprise auto-link;
   re-check `is_active` on the SSO existing-link path; reject `\` in `next`.
5. **Harden the hypervisor and sandbox boundary (H5, H7, M12).** Enable XO TLS
   verification; set the microVM FORWARD policy to DROP with scoped egress-only
   rules; pin and checksum-verify the VMM, kernel, and azcopy downloads.
6. **Add rate limiting and DoS bounds (M1, M2, M5, M6).** Per-IP/per-account
   throttling and lockout on auth endpoints; cap password length; bound and
   size-limit git request bodies and gzip decompression; return generic 5xx
   errors.
7. **Ship security headers and fix the client-side gate (M7, M8).** Add CSP /
   frame-ancestors / nosniff / referrer-policy on the nginx console and Firebase
   site; treat the pre-launch gate as UX only.
8. **Mobile release hardening (H8, H9, M15, M17).** Real release signing key;
   remove blanket cleartext + move to HTTPS/`wss://`; server-side auth for the
   app; enable R8 and stop distributing source maps.
9. **CI/supply-chain hygiene (M13, M14, L13, L14, L15, M16).** SHA-pin
   third-party actions; separate untrusted-build from secret-holding deploy jobs;
   scope `contents: write` to the release job; env-indirect tag values; pin the
   mailpit image; run `npm audit`/`cargo audit` in CI and clear the frontend
   advisories.
10. **Remaining low/informational items** as maintenance: constant-time CSRF
    compare, indexed token lookup, persistent SSH host key, backend `.gitignore`
    env patterns, cookie-consent withdrawal, LICENSE files, and the rest.
