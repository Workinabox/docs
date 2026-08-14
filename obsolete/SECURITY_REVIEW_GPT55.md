# Workinabox Security Review - GPT55

Date: 2026-07-04

## Scope and Method

This review covered the repositories present under `/home/fgos/projects/workinabox`:

- `.github`
- `app`
- `assets`
- `backend`
- `dev`
- `docs`
- `frontend`
- `iac`
- `website`

I explicitly did not read `docs/SECURITY_REVIEW_OPUS48.md`, per instruction.

This was a source, configuration, infrastructure, and dependency review. I excluded generated/vendor-heavy directories such as `.git`, `node_modules`, Rust `target`, Terraform plugin directories, and build outputs, except where generated artifacts or lockfiles were needed for security context. I reviewed tracked files, relevant local ignored configuration files, Dockerfiles, workflows, Terraform, deployment scripts, app manifests, API handlers, auth code, VM/sandbox code, frontend and mobile code, and docs. I also ran runtime `npm audit --omit=dev --json` for JavaScript apps where available. Rust advisory tooling (`cargo audit`/`cargo deny`) was not installed in the environment, so Rust dependencies still need a proper advisory pass in CI or a developer workstation.

No committed live secrets were found in tracked files by the targeted scans I ran. However, there are serious security issues in authorization coverage, local secret handling, VM launch controls, and deployment hardening.

## Executive Summary

The most important risk is that large parts of the backend JSON API are not protected by authentication or authorization. This appears to allow anonymous users to list, create, or update core objects such as meetings, organizations, projects, agents, boards, pipelines, work items, and some repository metadata/content routes. The git HTTP and SSH protocol paths are much better protected, but the JSON repository browsing paths appear to bypass the same visibility checks.

The second immediate risk is secret exposure in local ignored files. These are not committed, but this review process did read several non-placeholder-looking production or shared secrets in `dev/local/oidc.env` and `iac/terraform.tfvars`. Those values should be considered exposed to this tool session and rotated.

The third major risk is VM/sandbox control. Agent creation and activation routes appear unauthenticated, and activation can provision Firecracker VMs. VM template names are not sufficiently constrained before being used in host filesystem paths, and runtime jail/overlay files are created with broad permissions.

The system has some good foundations: passwords use Argon2id, tokens and sessions are hashed at rest, CSRF protection exists for cookie-authenticated unsafe requests, git protocol handlers mostly perform role checks, and most CI workflows use restrained default permissions. The main priority is to make authorization default-deny across the API, rotate secrets, and lock down VM/sandbox and deployment surfaces.

## Immediate Action Plan

1. Rotate secrets that were present in local ignored files: OIDC/Google client secrets, Resend keys, Xen Orchestra token, Azure SAS URLs, database password if reused outside disposable dev, and any bootstrap owner credentials.
2. Make the backend HTTP API default-deny. Every non-public route should require authentication and route-specific authorization.
3. Fix JSON repository APIs so private repository content cannot be read outside the same repo role/visibility rules used by git HTTP and SSH.
4. Ensure deactivated users cannot use personal access tokens or SSH keys.
5. Require authorization for agent activation/deactivation and all VM launch paths; allowlist VM templates and hard-limit VM resources.
6. Remove secret and token links from logs, including email bodies and transcript-like sensitive content.
7. Add rate limits, request size/time limits, security headers, dependency scanning, secret scanning, IaC scanning, and artifact verification/signing.

## Findings

### F-001: Live secrets in local ignored files

Severity: Critical

Affected areas:

- `dev/local/oidc.env`
- `iac/terraform.tfvars`
- `website/.env.local`
- `frontend/.env`
- `app/ios/.xcode.env`

What I found:

- `dev/local/oidc.env` contains non-placeholder-looking OAuth/OIDC client secrets, a Resend API key, and owner/dev credentials.
- `iac/terraform.tfvars` contains non-placeholder-looking Xen Orchestra and Azure deployment secrets, including SAS URLs with write/list capabilities, OIDC/Google secrets, Resend secrets, database password material, public IP configuration, and SSH key material.
- `website/.env.local` contains a Google Analytics measurement ID, which is public client configuration rather than a secret.
- `frontend/.env` contains `VITE_USE_STUB=true`, which is a configuration concern but not a secret.
- `app/ios/.xcode.env` did not contain a comparable secret.
- These sensitive files appear to be ignored and not committed, and a tracked-file secret scan found only placeholders or tests. Still, the live values were read during this review and should be treated as exposed.

Impact:

Compromise of any of these values could allow third-party API abuse, identity-provider impersonation, Azure blob access, infrastructure deployment access, email sending, or unauthorized app access depending on which value is used in shared or production environments.

Recommendations:

- Rotate every live secret present in the ignored files that was read during this review.
- Replace local plaintext secret files with a secret manager or encrypted-at-rest workflow such as SOPS/age, 1Password, Vault, cloud secret manager, or equivalent.
- Split Azure SAS URLs by purpose and environment. Prefer short-lived, least-privilege tokens, for example read-only for deploy and tightly scoped write for CI upload.
- Add pre-commit and CI secret scanning using tools such as gitleaks or trufflehog.
- Add a documented local secret bootstrap process so developers do not accumulate long-lived shared secrets in repo working trees.

### F-002: Backend JSON API has broad unauthenticated access

Severity: Critical

Affected areas:

- `backend/crates/wiab-inf/src/http_api.rs`
- Domain services called by those handlers

What I found:

Several handlers correctly call authentication or role checks, including user management, token management, repo creation, commit creation, and repo visibility changes. However, many other handlers in `http_api.rs` do not call `authenticate`, `require_owner`, `require_org_role`, or `require_repo_role`.

Routes/operations that appear unguarded include:

- Meetings: list/create routes and signaling upgrade path.
- Organizations: list/create/get/update.
- Projects: list/create/get/update.
- Agents: list/create/get/update/activate/deactivate.
- Boards: list/create/get/update.
- Repositories through JSON APIs: list/get/update plus branches/files/raw/commits reads.
- Pipelines: list/create/get/update.
- Work items: list/create/get/update, done-state operations, fulfill/unfulfill operations.
- WebSocket signaling via `/signal`.

Impact:

An anonymous user may be able to enumerate and mutate core business objects, read data, create or update agents, activate compute, create work, modify boards and projects, and join or observe meetings. This is a system-wide authorization failure, not just a missing guard on one route.

Recommendations:

- Introduce a route-level auth policy table or middleware layer where default behavior is deny.
- Require authentication for every non-public domain route.
- Enforce object-specific authorization for each operation: org owner/admin/member, project role, repo read/write/admin, and explicit meeting participant access.
- Add integration tests that prove anonymous users receive `401` or `403` for every non-public route.
- Add negative tests for authenticated users lacking the required role.
- Treat read routes as sensitive unless explicitly designed to be public.

### F-003: Private repository content can be exposed through JSON repo APIs

Severity: Critical

Affected areas:

- `backend/crates/wiab-inf/src/http_api.rs`
- `backend/crates/wiab-inf/src/git_http.rs`
- `backend/crates/wiab-inf/src/git_ssh.rs`

What I found:

The git HTTP and git SSH paths perform meaningful checks:

- Git HTTP allows anonymous reads only for public repositories and otherwise requires a token and role.
- Git SSH requires an SSH key identity and checks repo role.

The JSON repository paths do not appear to apply equivalent checks. Routes such as repository branch listing, file listing, raw file reads, commit listing, `get_repo`, `list_repos`, and `update_repo` are handled in `http_api.rs` without repo role enforcement.

Impact:

Private repository contents may be readable through JSON endpoints even when git clone/fetch is correctly denied. This bypasses the intended repository visibility model.

Recommendations:

- For repo read endpoints, call `require_repo_role(..., Operation::Read, ...)`.
- For repo mutation endpoints, require `Operation::Write` or `Operation::Administer` as appropriate.
- Preserve anonymous read only when `Visibility::Public` is explicitly true.
- Add tests that create a private repo and verify anonymous and unauthorized users cannot read branches, file trees, raw files, commits, or repo details.
- Keep git HTTP, git SSH, and JSON browsing paths backed by the same authorization helper to avoid future drift.

### F-004: Deactivated users can likely continue using PATs and SSH keys

Severity: High

Affected areas:

- `backend/crates/authbox-core/src/user_application_service.rs`
- `backend/crates/wiab-inf/src/http_api.rs`
- Git HTTP and SSH authentication paths

What I found:

`find_by_email` checks `user.is_active()` for password login. However, token and SSH-key resolution paths appear to iterate users and match token hashes or SSH key fingerprints without checking whether the matched user is still active.

The HTTP deactivation handler revokes browser sessions, but it does not revoke personal access tokens or remove/disable SSH keys.

Impact:

A deactivated user may lose web session access but retain machine/API/git access through personal access tokens or SSH keys. This undermines offboarding and incident response.

Recommendations:

- Require `user.is_active()` in token resolution and SSH fingerprint resolution.
- On user deactivation, revoke all sessions, revoke or disable personal access tokens, and disable or remove SSH keys.
- Add tests proving deactivated users cannot authenticate via password, session, PAT, git HTTP, or git SSH.
- Consider adding a database-level active/revoked state to tokens and keys instead of relying only on user status.

### F-005: Unauthenticated agent activation can launch VMs

Severity: High

Affected areas:

- `backend/crates/wiab-inf/src/http_api.rs`
- `backend/crates/wiab-core/src/agent_application_service.rs`
- `backend/crates/wiab-inf/src/firecracker_runtime.rs`

What I found:

Agent creation, updates, activation, and deactivation appear to be exposed through HTTP handlers without authentication guards. Agent activation can provision a Firecracker VM when running on a host configured with KVM and images.

Impact:

An anonymous caller may be able to create or modify agents and trigger compute provisioning. This can become a denial-of-service, cost, resource exhaustion, or isolation risk.

Recommendations:

- Require org/project admin authorization for creating, updating, activating, or deactivating agents.
- Add per-org and global concurrency limits for active agents and VMs.
- Rate-limit activation/deactivation endpoints.
- Audit-log VM launch and stop events with actor, org, project, agent, template, and resource size.
- Add tests proving anonymous and unauthorized users cannot launch or stop VMs.

### F-006: Firecracker template and filesystem handling needs hardening

Severity: High

Affected areas:

- `backend/crates/wiab-core/src/vm_domain.rs`
- `backend/crates/wiab-inf/src/firecracker_runtime.rs`
- `iac/images/*`

What I found:

`VmTemplate::new` rejects empty strings but does not restrict characters or enforce an allowlist. `FirecrackerRuntime` uses the template value in a path like `images_dir.join(format!("{}.ext4", spec.template))`. A template containing path traversal can cause path resolution outside the intended images directory, at least for image selection/probing.

The runtime creates per-VM jail roots with mode `0777` and overlay drive files with mode `0666` so a jailed UID can access them. This is convenient but broad on a shared host.

The base guest image creates a `wiab` user with passwordless sudo, optionally installs a build-time SSH public key, and the developer image installs Rust via a `curl | sh` rustup pipeline. These may be acceptable for a controlled demo image, but they are not production hardened.

Impact:

Weak template validation can turn a VM image selector into a host filesystem path selector. Broad host permissions increase the blast radius of a compromised local user or service. Guest passwordless sudo and shared SSH keys increase the blast radius inside launched VMs.

Recommendations:

- Replace free-form template names with an enum or allowlist such as `base` and `developer`.
- Enforce a strict template regex such as `^[a-z0-9_-]+$`.
- Canonicalize selected image paths and assert they remain under the configured images directory.
- Avoid `0777`/`0666` host permissions. Prefer creating files as the Firecracker jail UID/GID using a small privileged helper, `chown`, or tighter ACLs.
- Remove passwordless sudo and baked shared SSH keys from production guest images.
- Sign or attest rootfs/kernel/initramfs artifacts and verify them before launch.

### F-007: Meeting and SFU signaling is unauthenticated and ID-based

Severity: High

Affected areas:

- `backend/crates/wiab-inf/src/http_api.rs`
- `backend/crates/wiab-inf/src/sfu.rs`
- Meeting domain/application services

What I found:

The `/signal` WebSocket upgrade path does not appear to require authentication. The signaling code trusts client-supplied `meeting_id` and `participant_id` and validates only that the meeting is active and that the participant belongs to the meeting.

Meeting list/create endpoints also appear unauthenticated, making IDs and participant IDs easier to discover or mint.

Impact:

An unauthenticated user may be able to join active meetings, impersonate participants, create transports, produce or consume audio, and possibly end meetings if they can act as an owner participant.

Recommendations:

- Bind meeting participants to authenticated users or signed invite tokens.
- Require authentication at WebSocket upgrade or in the first message before any media action.
- Treat `participant_id` as an authorization target, not an identity proof.
- Add per-connection, per-meeting, and per-user transport/media limits.
- Avoid exposing active meeting IDs and participant IDs through unauthenticated list routes.

### F-008: Secrets and sensitive links are logged

Severity: High

Affected areas:

- `backend/crates/wiab-inf/src/bootstrap.rs`
- `backend/crates/authbox-inf/src/logging_email_sender.rs`
- `backend/crates/authbox-inf/src/resend_email_sender.rs`
- `backend/crates/wiab-inf/src/transcription.rs`
- `backend/crates/wiab-inf/src/vm_comms_broker.rs`

What I found:

The bootstrap path logs a default owner password and bootstrap token when seeding an empty store. The logging email sender logs password reset, invite, and verification links for dev convenience. The Resend email sender logs accepted email body content, which can include sensitive links even when a real provider is configured.

The transcription path logs transcript text, and the VM communications broker logs report lines received from guest agents.

Impact:

Logs can become an alternate secret store. Password reset links, invite links, verification links, transcripts, and agent output may be captured by log aggregation, shell history, support bundles, or cloud logging systems.

Recommendations:

- Never log reset, invite, verification, session, token, or email body content in production.
- Gate any dev-only secret printing behind an explicit environment variable such as `WIAB_BOOTSTRAP_PRINT_SECRETS=1`.
- Fail closed in production if no real email sender is configured instead of falling back to a logging sender.
- Log email message IDs/statuses, not email bodies.
- Redact transcript and agent output logs by default; provide explicit debug opt-ins for local development only.

### F-009: Infrastructure and Terraform state can retain secrets

Severity: High

Affected areas:

- `iac/main.tf`
- `iac/variables.tf`
- `iac/templates/provision.env.tftpl`
- `iac/scripts/provision.sh`
- `iac/scripts/wiab-deploy.sh`

What I found:

Terraform variables are marked sensitive in some places, which is good. However, secrets are still interpolated into provisioner commands, local/remote exec scripts, generated environment files, and trigger values. Terraform state can retain sensitive values even when CLI output redacts them.

`provision.sh` and `wiab-deploy.sh` source `/etc/wiab/provision.env` as shell. Cloud-init creates an `ubuntu` user with passwordless sudo, and deployment scripts rely on secrets and SAS URLs written to host files.

Impact:

Terraform state, provision logs, shell environments, process listings, and host files can leak deployment secrets. A compromised deploy user or leaked state file can become infrastructure access.

Recommendations:

- Store Terraform state in an encrypted remote backend with strict access controls.
- Avoid putting secrets in provisioner triggers or command strings.
- Prefer cloud secret manager retrieval at runtime over writing long-lived secrets to `/etc/wiab/provision.env`.
- If an environment file is needed, make it root-readable only and consume it via systemd `EnvironmentFile` or a non-evaluating parser, not shell `source`.
- Remove broad passwordless sudo after bootstrap or replace it with a dedicated limited deploy user.

### F-010: Missing rate limits, body limits, and abuse controls

Severity: Medium

Affected areas:

- Backend HTTP API
- Auth routes
- Git HTTP routes
- WebSocket/SFU routes

What I found:

I did not find application-level rate limiting, body-size limiting, request timeouts, login throttling, reset throttling, token creation throttling, git operation throttling, or SFU transport throttling. There is also a code comment noting a login timing oracle because Argon2 verification is only performed when a credential exists.

Impact:

Attackers can brute-force or spray credentials, enumerate accounts through timing, flood reset/invite flows, create excessive transports, upload large bodies, or abuse git/API operations.

Recommendations:

- Add a `tower` layer for request body limits and timeouts.
- Rate-limit login, signup, password reset, invite acceptance, email verification, OIDC callback, token creation, repo commit creation, git HTTP, and WebSocket operations.
- Use a dummy Argon2 hash path for unknown users to reduce login timing differences.
- Use generic auth error responses where practical.
- Add per-user, per-IP, and per-org quotas for expensive operations.

### F-011: Database schema relies heavily on application-only integrity

Severity: Medium

Affected areas:

- Backend migrations under `backend/crates/wiab-inf/migrations`
- Authbox migrations under `backend/crates/authbox-inf/migrations`

What I found:

The migrations intentionally avoid many foreign keys and constraints. I did not see strong database-level uniqueness or check constraints for several security-relevant concepts such as user email uniqueness, token hashes, SSH fingerprints, enum-like states, role values, scopes, and visibility values.

Impact:

Application bugs, races, manual changes, imports, or compromised write paths can create duplicate identities, orphaned records, invalid roles/states, or authorization ambiguity.

Recommendations:

- Add uniqueness constraints for normalized user emails, token hashes, and SSH fingerprints where globally unique identity is expected.
- Add foreign keys where lifecycle coupling is clear.
- Add check constraints for role, scope, state, visibility, and type columns.
- Add indexes around token hash and SSH fingerprint lookup paths.
- Keep application validation, but do not make it the only integrity boundary.

### F-012: Supply-chain and artifact provenance controls are incomplete

Severity: Medium

Affected areas:

- GitHub workflows across repos
- `iac/scripts/provision.sh`
- `iac/scripts/wiab-deploy.sh`
- `iac/images/*`
- Rust and JavaScript dependencies

What I found:

CI includes normal lint/test/build workflows, but I did not find consistent CodeQL/SAST, secret scanning, Rust advisory scanning, npm audit gates, OS/container scanning, Terraform scanning, SBOM generation, or artifact signing.

Deployment and image scripts download tooling and artifacts over HTTPS, including Firecracker, azcopy, GitHub release assets, and rustup. Some release downloads use adjacent SHA files, which is better than nothing, but these checksums are not equivalent to signed provenance.

Impact:

A compromised dependency, release asset, CDN, or workflow can affect deployed hosts, guest images, or application builds.

Recommendations:

- Add Dependabot or Renovate for all package ecosystems.
- Add `cargo audit` or `cargo deny`, `npm audit`, CodeQL, gitleaks, Trivy/Grype, and checkov/tfsec to CI.
- Verify Firecracker and azcopy checksums/signatures from trusted channels.
- Pin versions instead of using moving `latest` where production deployment is involved.
- Generate SBOMs and sign release artifacts with Sigstore or GitHub artifact attestations.
- Verify guest image signatures before the backend launches them.

### F-013: JavaScript dependency advisories found

Severity: Medium

Affected areas:

- `frontend/package-lock.json`
- `app/package-lock.json`
- `website/package-lock.json`

What I found:

`npm audit --omit=dev --json` results:

- `frontend`: 1 high runtime vulnerability. Transitive `form-data` is affected by a CRLF injection issue in multipart field names/filenames (`GHSA-hmw2-7cc7-3qxx`, affected range `>=4.0.0 <4.0.6`). A fix is available.
- `app`: 1 moderate runtime vulnerability. Transitive `js-yaml` is affected by a quadratic-complexity denial-of-service issue through merge aliases (`GHSA-h67p-54hq-rp68`, affected `<3.15.0`). A fix is available.
- `website`: 0 runtime vulnerabilities from this audit.

Rust advisory status:

- `cargo audit` and `cargo deny` were not available in this environment, so Rust dependencies were not checked against RustSec advisories.

Recommendations:

- Update the dependency chains that pull in vulnerable `form-data` and `js-yaml`.
- Add runtime dependency audit gates to CI.
- Add RustSec advisory scanning to backend/dev CI.
- Track both runtime and dev dependencies separately; dev dependency compromise still matters in CI.

### F-014: Mobile app is configured for local development rather than production security

Severity: Medium

Affected areas:

- `app/src/backendConfig.ts`
- `app/android/app/src/main/AndroidManifest.xml`
- `app/android/app/build.gradle`
- `app/.github/workflows/release.yml`
- iOS app manifests

What I found:

The mobile app defaults to local `http://` and `ws://` endpoints. Android allows cleartext traffic. The release build signs with a generated debug-style keystore in the workflow, and release minification/obfuscation is disabled. Android backup is disabled, which is good. iOS disallows arbitrary loads while permitting local networking, which is appropriate for local development.

Impact:

If these settings are used outside local development, traffic can be intercepted, release provenance is weak, and reverse engineering is easier.

Recommendations:

- Use `https://` and `wss://` for production builds.
- Restrict cleartext traffic to debug/local builds through network security config.
- Use real release signing, Play App Signing, or an equivalent controlled signing process.
- Enable R8/ProGuard for release builds where compatible.
- Ensure production builds do not display or leak backend endpoint details unnecessarily.

### F-015: Frontend hardening gaps

Severity: Medium

Affected areas:

- `frontend/src/pages/LoginPage.tsx`
- `frontend/src/pages/UsersPage.tsx`
- `frontend/nginx.conf`
- `frontend/Dockerfile`
- `iac/templates/nginx-wiab.conf.tftpl`

What I found:

The frontend has a solid pattern for cookie-based auth with HttpOnly sessions and CSRF tokens. However:

- The `next` query parameter used after login should be sanitized client-side to a same-origin relative path. Some backend OIDC return paths are sanitized, but a crafted login URL can still influence client navigation.
- One-time personal access tokens are displayed in the DOM until the page state changes. This is operationally necessary, but sensitive.
- Nginx configurations lack common hardening headers such as CSP, HSTS, `X-Content-Type-Options`, `Referrer-Policy`, `Permissions-Policy`, and `frame-ancestors`.
- The container image appears to run nginx in its default root-oriented configuration.

Impact:

Missing headers increase XSS/clickjacking/data-leak blast radius. Unsanitized redirect targets can create confusing navigation or possible open-redirect-style behavior. Long-lived display of PATs increases exposure through screenshots, extensions, shoulder surfing, or XSS.

Recommendations:

- Sanitize `next` to a path beginning with `/` and reject protocol-relative or absolute URLs.
- Clear one-time PAT display after a short timeout or after copy, and never persist it.
- Add CSP, HSTS, `X-Content-Type-Options: nosniff`, `Referrer-Policy`, `Permissions-Policy`, and `frame-ancestors 'none'` or an intentional allowlist.
- Run nginx as a non-root user in containerized deployment where practical.

### F-016: Website and Firebase Hosting hardening gaps

Severity: Medium

Affected areas:

- `website/firebase.json`
- `website/src/firebase.ts`
- `website/src/components/LaunchGate.tsx`
- `website/src/components/AuthModal.tsx`
- `website/src/analytics.ts`

What I found:

Firebase client config is public by design, and the README correctly treats client-side launch gating as not being a security boundary. Analytics only loads after consent, which is good.

Firebase Hosting currently sets cache headers but does not appear to set CSP, HSTS, referrer, permissions, frame, or content-type hardening headers. Preview/launch gating is client-side only.

Impact:

If any private or pre-release content is assumed to be protected by the client gate, it can be accessed by bypassing JavaScript. Missing headers increase frontend exploit blast radius.

Recommendations:

- Keep all private content out of the client bundle and Firebase Hosting public assets.
- Enforce Firebase Auth, Firestore, and Storage Security Rules server-side.
- Restrict Firebase API keys by HTTP referrer and allowed domains where applicable.
- Add hosting security headers, including a CSP compatible with Firebase and GA.
- Continue to treat launch gating as UX only.

### F-017: Network exposure and SFU port range should be narrowed

Severity: Medium

Affected areas:

- `iac/scripts/provision.sh`
- `iac/templates/nginx-wiab.conf.tftpl`
- `backend/crates/wiab-inf/src/sfu.rs`
- `dev/local/docker-compose.yml`

What I found:

The backend and git SSH services bind to all interfaces in some configurations, while UFW and Nginx are intended to be the public control points. UFW opens a wide UDP range for SFU/media. The SFU code does not appear to set a narrow mediasoup port range. Local Docker Compose publishes Postgres, backend, git SSH, Mailpit, and OIDC mock ports broadly by default.

Nginx proxies to the backend with `proxy_ssl_verify off`, which is understandable for same-host self-signed localhost traffic but should not become a remote trust pattern.

Impact:

Wide binding and broad UDP ranges increase exposure. Local dev services may be reachable on a LAN. A misordered provisioning step or firewall change can accidentally expose backend ports directly.

Recommendations:

- Bind backend HTTP and git SSH listeners to loopback when Nginx is the intended public entry point.
- Narrow mediasoup UDP port range and open only that range.
- Bind local Docker Compose ports to `127.0.0.1` unless LAN access is intentional.
- Add Nginx rate limits and access logs for sensitive routes.
- Prefer Unix sockets or loopback-only backend TLS for same-host proxying.

### F-018: Password and account policy is minimal

Severity: Medium

Affected areas:

- Auth HTTP handlers
- Authbox domain/application services

What I found:

Passwords appear to require only a minimum length of 8 characters in several flows. I did not find breached-password screening, MFA support, account lockout/backoff, or SSO domain/group policy enforcement beyond basic OIDC configuration.

Impact:

Weak or reused passwords are more likely to succeed in credential-stuffing attacks, especially without rate limits or MFA.

Recommendations:

- Increase minimum password length to at least 12 characters.
- Add zxcvbn-style strength checks or breached-password checks.
- Add MFA or enforce SSO for privileged roles.
- Add lockout/backoff and alerting for repeated failures.
- Require re-authentication for sensitive actions such as token creation, email changes, and role changes.

### F-019: OIDC account linking policy needs stricter tenant/domain validation

Severity: Medium

Affected areas:

- Authbox OIDC services and docs
- IaC OIDC variables

What I found:

Google OIDC appears to require verified email. Enterprise OIDC flows appear to treat the enterprise IdP as authoritative and may auto-link by email without requiring an `email_verified` claim. That can be acceptable for a tightly controlled enterprise issuer, but it is risky if the issuer, tenant, or domain policy is misconfigured or broad.

Impact:

A misconfigured or multi-tenant OIDC provider could allow an external identity with a matching email to link into an existing account.

Recommendations:

- Validate OIDC config at startup when enabled: non-empty issuer, client ID, client secret, redirect URI, and allowed domains/tenants.
- For enterprise providers, require issuer and tenant allowlists.
- Consider requiring verified email, allowed domain, or group membership claims before auto-linking.
- Log account-link events without logging tokens or claims that contain sensitive data.

### F-020: CSRF implementation is useful but cannot compensate for missing auth

Severity: Medium

Affected areas:

- `backend/crates/wiab-inf/src/http_api.rs`
- Auth session/CSRF services

What I found:

The CSRF design uses an HttpOnly session cookie and a separate CSRF cookie/header pattern backed by server-side session state. That is a good foundation. However, the guard only protects cookie-authenticated unsafe requests when a valid session is present. For routes that do not require authentication, the CSRF guard does not provide meaningful protection.

I also did not find explicit Origin/Referer validation for unsafe cookie-authenticated requests.

Impact:

CSRF controls may be bypassed on unguarded state-changing routes because those routes are already anonymously callable. Cookie-authenticated routes would benefit from Origin checks as an additional browser-side boundary.

Recommendations:

- First fix missing route authentication and authorization.
- Add Origin/Referer validation for unsafe cookie-authenticated requests.
- Keep CSRF exemptions narrow and documented.
- Add tests for missing, invalid, and cross-origin CSRF attempts.

### F-021: Model and file path configuration should reject traversal

Severity: Medium

Affected areas:

- `backend/crates/wiab-inf/src/model_paths.rs`
- `backend/crates/wiab-inf/src/llama_runtime.rs`
- `backend/crates/wiab-inf/src/llama_meeting_intelligence.rs`

What I found:

Model paths are constructed from configured filenames under the data directory. The code should treat configured model filenames as basenames and reject path traversal or absolute paths.

Impact:

This is primarily a privileged-configuration risk rather than a remote user risk. Still, a misconfigured environment value can make the service attempt to load files outside the intended models directory.

Recommendations:

- Require model file settings to be basenames, not paths.
- Canonicalize resolved paths and assert they remain under the configured models directory.
- Whitelist expected model extensions.
- Fail closed at startup when model paths are invalid.

### F-022: Local dev stack exposes services broadly

Severity: Low to Medium

Affected areas:

- `dev/local/docker-compose.yml`
- `dev/local/README.md`

What I found:

The local stack publishes Postgres, backend HTTP, git SSH, Mailpit, and mock OIDC ports. It also mounts local data directories. This is fine for local-only work, but the port mappings appear to bind broadly unless Docker is configured otherwise.

Impact:

On a shared network, local dev services may be reachable by other machines. With default dev credentials and local data mounts, that can expose source, mail previews, database contents, or local model/data files.

Recommendations:

- Bind local dev ports to `127.0.0.1`.
- Keep default credentials local-only and documented as unsafe.
- Avoid mounting broad home-directory paths into containers.
- Add a dev preflight warning when services bind to non-loopback interfaces.

### F-023: Release tooling and organization automation could be more constrained

Severity: Low to Medium

Affected areas:

- `dev/src/*`
- GitHub workflows
- Release docs/scripts

What I found:

The release tooling uses a GitHub token from the environment and performs useful checks for clean/synced repos. I did not see signed tag enforcement, release provenance, or a requirement that CI has passed immediately before tagging. Some workflows have write permissions for release steps, which is expected, but should be tightly limited.

Impact:

A leaked broad token or compromised release workstation can publish tags and releases across repos. Unsigned tags make provenance weaker.

Recommendations:

- Use fine-grained PATs or GitHub App credentials with least privilege.
- Sign release tags and artifacts.
- Require green CI and protected branches before release tagging.
- Add artifact attestations and verify them during deployment.

## Positive Controls Observed

- Password hashing uses Argon2id.
- Session, token, reset, and verification secrets are stored as hashes rather than plaintext.
- Cookie sessions are HttpOnly and become Secure when HTTPS base URLs are configured.
- CSRF protection exists for cookie-authenticated unsafe requests.
- Git HTTP and git SSH protocol paths contain meaningful repository role/visibility checks.
- User management and some repo mutation routes already enforce owner or repo roles.
- Several GitHub workflows use restrained default permissions such as `contents: read`.
- Firebase preview deploy workflows avoid unsafe fork-secret behavior.
- Android backup is disabled.
- iOS arbitrary network loads are not globally allowed.
- Tracked-file scans did not identify committed live secrets, only placeholders/tests/examples.

## Recommended Fix Order

1. Secret rotation and logging cleanup.
2. API authorization default-deny and route tests.
3. Repository JSON API authorization parity with git HTTP/SSH.
4. Deactivated user PAT/SSH revocation.
5. Agent/VM authorization, template allowlisting, and VM resource limits.
6. Meeting/WebSocket authentication and invite-token model.
7. Rate limits, body limits, Origin checks, and password policy hardening.
8. Terraform/deploy secret handling and host hardening.
9. Dependency, secret, IaC, and artifact scanning in CI.
10. Frontend, website, mobile, and Nginx security header hardening.

## Notes on Verification Gaps

- This review did not perform live exploitation against a running environment.
- Rust dependency advisories still need a real `cargo audit` or `cargo deny` run.
- Some findings depend on route wiring and runtime configuration. The missing auth patterns are clear enough to treat as high-confidence, but each route should still be covered by integration tests after fixes.
- Local ignored secrets were inspected only enough to identify risk categories; exact values are intentionally not included in this report.
