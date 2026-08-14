# Security Review — Workinabox

Date: 2026-07-04
Amended: 2026-07-10 — `cargo-audit` was subsequently installed and run against
both Rust workspaces; results and remedies were folded into the "Dependency and
supply-chain audit" section (new "Rust dependency audit (`cargo-audit`)"
subsection) and I8 was updated accordingly. No C/H/M/L finding IDs changed.
Amended: 2026-07-11 — C1 (unauthenticated state-changing REST endpoints) was remediated in the
`backend` repo and is now marked **✅ Fixed** in its finding entry; the entry is retained for the
historical record. No finding IDs changed.
Amended: 2026-08-03 — second remediation batch. Every remaining backend code finding was
re-verified against the current tree (which has moved: config centralisation, the Docker VM
runtime, teams, NATS) and then fixed: **C2, H1, H2, H3, H4, M1, M2, M3, M5, M6, L4, L5, L6**
are now marked **✅ Fixed**. Entries are retained in full for the historical record. Two
findings' scope changed on inspection and this is noted in their entries. Infrastructure
(`iac`) and mobile (`app`) findings are untouched and remain open. No finding IDs changed.
Amended: 2026-08-03 (later) — third remediation batch: **M4** and every Low in `backend`,
`frontend`, `website` and `dev` are now **✅ Fixed**, along with the dependency advisories in
both Rust workspaces. Two entries record decisions rather than fixes (**L8**, and **M8**/**L11**
as knowingly accepted). One entry is new: **R1**, a regression introduced by the second batch
and found while verifying this one. `app` and the rest of `iac` remain untouched.
Amended: 2026-08-04 — **C3 re-assessed and its severity reduced.** The original entry instructed
the reader to treat the Terraform state as already exposed and rotate six credentials. That was
asserted without establishing where the state lived or who could reach it. It was never
committed, is absent from the project tree, and lives on one operator machine. The structural
gap (no encrypted, locked remote backend) stands and now also names the state-loss and
concurrent-apply risks, which are the stronger arguments for fixing it. Cross-references in the
summary, the secret inventory, M11 and the roadmap were corrected to match. No IDs changed.
Amended: 2026-08-04 (later) — fourth remediation batch: **H6, H7, M7, M9, M10, M12, M13, M14,
L19, L20** fixed. That closes every finding outside `app` and the deferred items. `iac` changes
take effect on the operator's next provision; they are code, not applied state.
Amended: 2026-08-09 — **first deployment.** The demo host was rebuilt from scratch
(`terraform destroy` + `apply`) on backend `v0.1.16` / frontend `v0.1.10`, so every `iac` and
code fix above is now *applied*, not just committed. Operator items 1, 2 and 5 are done: R1's
nginx `X-Forwarded-For` overwrite is live and verified (a rotating spoofed header still hits the
429), the breaking tfvars changes were made, and the Postgres password was rotated by the fresh
provision (M9). H6 (`wiab.env` `0640 root:wiab`), H7 (FORWARD `DROP`) and the microVM smoke boot
were verified on the host. The from-scratch path had never run since the M11 change and exposed
three `iac` deploy bugs, all fixed (see `iac` PR #19). **H5 stands, and by an operator decision
is accepted for now** (`xoa_insecure` stays `"true"`; a real XO certificate is scheduled
separately). Item 4 (website CSP) is unchanged.
Amended: 2026-08-09 (later) — **TLS restored.** Removing the public port-80 forward had broken
certbot's HTTP-01 challenge, leaving the demo HTTP-only (which blocks registration — the backend
sets `Secure` cookies over its `https://` base URL). Fixed by switching to the **DNS-01**
challenge (no inbound port): a real Let's Encrypt certificate for `demo.workinabox.ai` is
installed and serving (443 + 80→301 redirect verified; owner login holds its Secure cookie).
DNS is at one.com (no API), so issuance/renewal are manual via the new `wiab-cert` helper — cert
expires 2026-11-07 and does **not** auto-renew. See `iac` PR #20. Reaching the host by name on
the LAN uses a per-machine hosts entry (`192.168.1.4 demo.workinabox.ai`), documented in the iac
README.
Amended: 2026-08-10 — **C3 verification block corrected.** Its "Verified 2026-08-04:
no tfstate/tfvars exist" bullet was invalidated by the 2026-08-09 apply (recorded in the
amendment above), which wrote `terraform.tfstate`, `.backup` and `terraform.tfvars` into `iac/`.
Re-verified against the tree: the files now exist, still git-ignored, still never committed, and
there is still no remote backend. Severity unchanged (local + git-ignored); the "files don't
exist" basis is removed. No IDs changed. (Part of the workspace-wide docs-truth pass — see
`DOCS_AUDIT.md`.)
Reviewer: automated multi-agent security audit (Claude Code)
Scope: all nine repositories in the `Workinabox` GitHub organisation
Method: full source read of every repo plus targeted static analysis; every
Critical and High finding was independently re-verified against the source.

## State of play (2026-08-04)

Four remediation batches have been applied across `backend`, `frontend`, `website`, `dev`,
`iac` and this repository. **No code finding outside `app` remains open.** Nothing was
deployed at any point — every `iac` change is committed code that takes effect on the next
provision.

Read this section first if you are picking the work up. The finding entries below are the
authority on *what* and *why*; this is the shortlist of what is still outstanding and who can
act on it.

### Outstanding — only the operator can do these

| # | Action | Why it matters |
|---|--------|----------------|
| 1 | ~~Re-provision for R1~~ **Done 2026-08-09** | Per-IP rate limiting is live and verified — a rotating spoofed `X-Forwarded-For` still hits the 429. |
| 2 | ~~Update `terraform.tfvars`~~ **Done 2026-08-09** | Version pins, strong `db_password`, `owner_password`, `vm_images_version = "v2"` all set; `plan` is clean. |
| 3 | **`xoa_insecure` is `"true"` — accepted for now** | H5 open by operator decision (2026-08-09). Needs a real certificate on Xen Orchestra, then flip to `"false"`. |
| 4 | **Flip the website CSP from `Content-Security-Policy-Report-Only` to `Content-Security-Policy`** | It ships in report-only because the policy is derived from source but was never exercised in a browser. Load a page, confirm no console violations, then enforce. `website/firebase.json`. |
| 5 | ~~Rotate the live Postgres password~~ **Done 2026-08-09** | The from-scratch rebuild created the DB fresh with the new strong password (M9). |
| 6 | ~~Restore TLS on the demo host~~ **Done 2026-08-09** | Switched to the DNS-01 challenge (no inbound port). Real Let's Encrypt cert installed and serving; 443 + redirect + Secure-cookie login verified. Manual issuance/renewal via `wiab-cert` (one.com has no API); **expires 2026-11-07, no auto-renew**. See `iac` PR #20. |

### Breaking configuration changes

`terraform.tfvars` needs attention before the next `apply`:

- `db_password` — **no longer has a default** and is validated: at least 16 characters from
  `[A-Za-z0-9._~-]`. The charset is deliberate; the value lands in a `DATABASE_URL` and in a
  shell-sourced env file.
- `ssh_admin_cidr` — **new**, optional. Unset means SSH accepts connections from any source,
  and provisioning logs a warning saying so. Set it to your network.
- `rtc_min_port` / `rtc_max_port` — **new**, default `40000`–`40999`. Must match the backend's
  `WIAB_MEDIASOUP_MIN_PORT`/`MAX_PORT`; the firewall opens exactly this range.

`terraform.tfvars.example` carries all three with comments.

### One ordering constraint

The backend pins its WebRTC transport range and the firewall opens exactly that range. The
backend change must be live **before or with** the narrowed firewall rule. Both are merged, so
this only matters if you deploy them separately: a mismatch presents as media that negotiates
successfully and then carries no audio, which is an unpleasant thing to debug.

### The decision that blocks six findings

**H8, H9, M15, M17, L17 and L18 are all in `app`, and all wait on one architectural choice:**
how the mobile client authenticates. Three options were weighed:

1. **The app performs OIDC against the already-configured IdP (Google/Entra) and the backend
   exchanges the resulting `id_token` for a session.** Recommended. Uses
   `react-native-app-auth` for authorization-code + PKCE in the system browser, reuses the
   JWKS/nonce validation the backend already has, and needs one new endpoint. No password is
   ever typed into the app, and SSO and MFA work.
2. **The backend becomes an OAuth 2.0 authorization server** (`/oauth/authorize`, `/oauth/token`,
   rotating refresh tokens). The fully correct, IdP-independent answer, and a substantial
   feature in its own right.
3. **Password grant against the existing `/auth/session`.** Fastest, and deprecated in OAuth
   2.1 for good reasons — it cannot do SSO or MFA and would have to be migrated off.

Whichever is chosen, **H9 must land in the same work**: the app currently uses `http://` and
`ws://`, and a bearer token over cleartext is worthless. Note the app cannot reach the backend
today regardless — it calls `GET /meetings`, a route that moved under
`/organizations/{id}/meetings`, and the signalling socket now requires authentication.

### Decisions already taken

Recorded so they are not silently reopened. Each has its reasoning in the relevant entry.

- **M1's progressive per-account lockout** — deliberately not implemented. Per-IP limiting plus
  an Argon2 concurrency cap covers the exploitable part; lockout is itself a denial-of-service
  vector against a known username.
- **M8, L8, L11** — accepted, not fixed.
- **C3** — deferred. Re-assessed 2026-08-04: not an active leak.
- **M16's breaking upgrades** — not taken. `vite` needs `--force` past an exact pin and
  `react-router` needs a major; both are build/routing changes the test suites would not
  necessarily catch.
- **M12, M13** — partial by design. See their entries for what is still unverified.

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

### Remediation status (2026-08-04, after four batches)

| | Fixed | Accepted, documented | Open |
|---|:--:|:--:|:--:|
| Critical | C1, C2 | — | C3\* |
| High | H1, H2, H3, H4, H6, H7 | — | H5\*, H8, H9 (`app`) |
| Medium | M1†, M2, M3, M4, M5, M6, M7, M9‡, M10, M11, M12§, M13, M14 | M8 | M15, M16†, M17 (`app`) |
| Low | L1–L7, L9, L10, L12–L16, L19, L20 | L8, L11 | L17, L18 (`app`) |
| Regression | R1\* | — | — |

\* Needs an action only the operator can take: C3 is a backend migration decision (re-assessed
2026-08-04 — not an active leak); H5 needs a real certificate on Xen Orchestra before
`xoa_insecure` can be `"false"`, and the repo default is already `"false"`; R1's fix is merged
but takes effect on re-provision.

† M1 is complete except progressive per-account lockout, deliberately deferred — it is itself a
denial-of-service vector against a known username. M16 is reduced, not closed: the non-breaking
advisories are cleared in both web repos; what remains needs a `vite --force` and a
`react-router` major.

‡ M9's validation is in place; rotating the live password is the operator's step.
§ M12 covers the privileged artifacts (firecracker/jailer, nats-server). azcopy and the
smoke-test images are still unverified — see the entry.

**Everything still open is in `app`, or needs the operator.** No code finding outside `app`
remains. The `app` findings — H8, H9, M15, M17, L17, L18 — are all blocked behind native
authentication, which is a feature rather than a fix: it means OAuth 2.0 authorization-code +
PKCE through the system browser, and this backend is an OIDC *relying party*, not a provider,
so there is no `/oauth/authorize` for a native client yet.

All `iac` changes are code and take effect on the next provision. Nothing in this remediation
was deployed.

### Finding counts by severity

| Severity | Count |
|----------|:-----:|
| Critical | 3 |
| High | 9 |
| Medium | 17 |
| Low | 20 |
| Info | 10 |

### Top risks to fix first

Original ordering, kept as written — 1 and 3 are now done; see the status table above.

1. ~~**C1/C2 — Unauthenticated REST API.**~~ Done (2026-07-11 and 2026-08-03).
2. **C3 — Plaintext secrets in local Terraform state and `tfvars`.** Move to an
   encrypted remote backend and rotate every secret those files contain. **Now the
   highest remaining risk.**
3. ~~**H1 — `owner/owner` default admin plus plaintext token logged at boot.**~~ Done
   (2026-08-03).
4. **H5/H6/H7 — Infrastructure exposure.** Xen Orchestra TLS verification is
   disabled, `wiab.env` is world-readable, and untrusted microVMs can pivot to
   the host and LAN. **Now the highest remaining cluster after C3.**

## Scope

| Repo | Stack | Role |
|------|-------|------|
| `backend` | Rust (axum, tokio, sqlx/tokio-postgres, russh, git2, Firecracker) | API, auth/identity (`authbox`), git hosting, SFU, agent sandbox |
| `frontend` | React + Vite + TypeScript, nginx, Docker | Frontend (admin/management UI) |
| `website` | React + Vite + TypeScript, Firebase Hosting | Public marketing site |
| `app` | React Native (bare) | Mobile audio/meeting client |
| `iac` | Terraform (xenorchestra), cloud-init, shell | XCP-ng provisioning + deploy |
| `dev` | Rust CLI | `monitor` / `release` tooling |
| `docs` | Markdown | Planning + architecture docs (this report lives here) |
| `assets` | Images + a static handoff page | Visual identity |
| `.github` | Markdown | Org profile / architecture notes |

## Critical findings

### C1 — Unauthenticated state-changing REST endpoints across every tenant

**Status: ✅ Fixed (2026-07-11)** — a global fail-closed authentication middleware
(`require_authentication` in `crates/wiab-inf/src/http_api.rs`, sibling to `csrf_guard`) now gates
every route except a small public allow-list (`/health`, `/auth/*`, and the git Smart-HTTP
endpoints, which self-authenticate). On top of that, every state-changing handler enforces
per-resource authorization (`require_org_role` / `require_repo_role` / `require_owner`). Meetings,
which previously belonged to no tenant, were made org-scoped
(`/organizations/{organization_id}/meetings`) so they authorize like every other resource. The
finding is retained below for the historical record.

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

**Status: ✅ Fixed (2026-08-03)** — C1's middleware closed the anonymous half of this in July,
but the authorization half survived: the six browse handlers still took no headers and never
consulted `Visibility` or role, so *any* authenticated caller — including a role-less signup
account — could read every private repository in every organization. A
`require_repo_read_access` helper now applies the git transport's rule verbatim (Public is
readable by any authenticated caller, Private needs a Read role) and `list_repos` uses the
existing project→org check. Regression tests drive the real router.

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

### C3 — No encrypted, locked remote Terraform state (severity reduced 2026-08-04)

**Status: ⚠️ Open, but re-assessed — this is not an active leak.** The original entry told the
reader to treat the state as already exposed and rotate six credentials. That instruction was
based on the files being present at review time, without establishing where they lived or who
could reach them.

Re-verified 2026-08-10 (the 2026-08-04 block below said these files did **not** exist; that was
true then but the 2026-08-09 first deployment — `terraform destroy` + `apply` — created them, so
the earlier bullet is corrected here):

- `iac/terraform.tfstate` (68 KB), `iac/terraform.tfstate.backup` (61 KB) and
  `iac/terraform.tfvars` (5 KB) **now exist** in the working tree on the operator's machine,
  written by the 08-09 apply. They contain real secrets (state holds resource attributes;
  tfvars holds the DB password, Resend key, SAS URLs).
- They have still **never** been committed on any branch — `.gitignore` covers `*.tfstate`,
  `*.tfstate.*`, `*.tfvars`, `*.tfvars.json`, `tfplan`; only `terraform.tfvars.example`
  (no values) is tracked.
- The structural gap is unchanged: `versions.tf` has no `backend`/`cloud` block, so state is a
  local file only — no encryption at rest, no locking, no remote history.
- CI never touches state or secrets — `ci.yml` runs `fmt`, `init -backend=false` and `validate`.

The severity reasoning still holds: the files are local and git-ignored, so this is exposure
*only* if the operator's machine syncs them off (unencrypted disk, OneDrive scope, backup) — a
question about that machine, and on WSL specifically about whether the checkout sits under
`/mnt/c/...` (Windows/OneDrive scope) or the WSL2 native filesystem. But "the files don't exist"
is no longer a valid basis for that reasoning, and the state-loss / concurrent-apply risks (a
single un-backed-up local state file) are now the stronger arguments for a remote backend.

**Rotation is therefore not indicated by this finding alone.** It becomes indicated if the
state has been within reach of something that copies files off the machine — an unencrypted
disk, a cloud-synced folder, or a backup — which is a question about that machine, not about
this repository. On WSL the distinction that matters is whether the checkout sits under
`/mnt/c/...` (Windows filesystem, inside whatever OneDrive/backup scope the profile has) or on
the WSL2 native filesystem (`~/...`, whose ext4 image OneDrive does not sync by default).

- **Repo:** `iac`
- **Location:** `versions.tf:1-10` — no `backend`/`cloud` block anywhere; state defaults to a
  local file.
- **Issue that remains:** Terraform state is written to the operator's disk unencrypted and
  unlocked. By field name (values were never extracted), a populated state contains
  `db_password`, `google_client_secret`, `oidc_client_secret`, `resend_api_key`, and the
  rendered cloud-init embedding the DB password and the Azure blob **SAS tokens** for
  `WIAB_MODELS_URL` / `WIAB_IMAGES_URL`. A populated `terraform.tfvars` additionally holds
  `xoa_token`, the Xen Orchestra API token. The `sensitive = true` flags only mask CLI output;
  they do not encrypt state, and provisioner `triggers` store values unmasked regardless.
- **Impact, restated honestly:** three distinct risks, only one of which is about disclosure.
  1. *Confidentiality* — anything that reads that machine's disk (theft, malware, an
     unintended sync or backup) obtains the hypervisor control-plane token, the OAuth/OIDC
     client secrets, the email-provider key, the DB password, and long-lived SAS URLs. Bounded
     by the security of one machine rather than by anything in this repository.
  2. *Availability* — the state is the only record mapping this configuration to the live
     XCP-ng resources, and it exists in exactly one place with no backup. Lose the WSL distro
     and Terraform no longer knows the VM exists; the next `apply` tries to create a second one
     and recovery is `terraform import`, by hand.
  3. *Integrity* — no locking, so two concurrent applies can corrupt the state. Low probability
     with one operator; not zero across two machines.
- **Recommendation:** Move to a backend with encryption at rest, locking and versioned history
  (Terraform Cloud, `azurerm` blob — this project already uses Azure storage for the VM images
  — or S3 + DynamoDB). That addresses all three risks at once, and is the reason to do it
  rather than the disclosure argument alone. Keep secrets out of `triggers` and rendered
  templates; source them at apply time. Rotation is a decision to make from the state of the
  operator's machine, not a blanket action implied by this finding.

### R1 — Per-IP rate limiting was bypassable in production (regression from this remediation)

- **Repo:** `iac` (introduced by the `backend`/`frontend` changes of 2026-08-03)
- **Status: ✅ Fixed (2026-08-03)** — `scripts/provision.sh` now sets
  `X-Forwarded-For $remote_addr`. **Requires a re-provision (or an equivalent edit to
  `/etc/nginx/sites-available/wiab` plus an nginx reload) to take effect.**
- **Issue:** M1's rate limiter keys on `X-Forwarded-For` and takes the first address in the
  list. The batch-2 work hardened `frontend/nginx.conf` to overwrite that header — but that
  file only configures the local docker-compose stack. Production nginx is written by
  `iac/scripts/provision.sh`, which used `$proxy_add_x_forwarded_for`: it *appends* the real
  address to whatever the client sent, leaving a client-chosen value in front.
- **Impact:** A client sending `X-Forwarded-For: 9.9.9.9` was rate-limited as `9.9.9.9`, and by
  rotating the value could make unlimited login attempts. For the interval between the two
  batches, M1 read as closed while providing no protection in production — which is worse than
  the original finding, because it invites you to stop worrying about brute force.
- **Why it is recorded here:** the fix and the mistake are both part of this remediation's
  history, and a reader of the M1 entry alone would otherwise believe rate limiting was working
  from 2026-08-03. It also generalises: this codebase has *two* nginx configurations, and a
  change to one is not a change to the other.

## High findings

### H1 — Default `owner/owner` admin and plaintext credentials logged at first boot

**Status: ✅ Fixed (2026-08-03)** — the password no longer has a baked-in default; seeding
refuses to invent one unless the advertised base URL is loopback, so a deployed backend fails
to start and names the variable to set rather than booting with a known administrator. The log
line names only the account: neither the password nor the token is logged. The bootstrap token
now expires in an hour and is written to `WIAB_BOOTSTRAP_TOKEN_FILE` at mode 0600 if a path is
given, and otherwise simply expires unused.

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

**Status: ✅ Fixed (2026-08-03)** — fixed at the root rather than at the gate. A
`MeetingParticipant` named a person but pointed at nobody, so a seat was something you could
*name*; it now carries the `UserId` entitled to occupy it, with aggregate invariants (a human
seat has a user, an agent seat does not, no user holds two) making the reverse lookup total.
`JoinMeeting` no longer accepts a `participant_id` at all — `validate_join` takes the
authenticated user and returns the seat they hold. The `/signal` upgrade resolves the caller
before upgrading, and `list_meetings` is gated on an organization role, closing the identifier
disclosure. The other signals needed no change: each requires a peer that only a validated join
creates, and `EndMeeting` was already owner-gated inside the aggregate.

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

**Status: ✅ Fixed (2026-08-03)** — the enterprise connection now requires `email_verified`.
The vulnerability was the *combination* with `auto_link_verified_email`, so this is the safe
half to enforce: an enterprise IdP that genuinely omits the claim must not auto-link. A test
that had encoded the old behaviour as intended ("enterprise accepts an unverified email") was
replaced by one asserting the rejection.

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

**Status: ✅ Fixed (2026-08-03)** — `UserDirectory` gained `may_authenticate`, answered from
the user's lifecycle state, and the existing-link branch now rejects a user who is not active.
That branch never consulted the directory by email, which is precisely why the `find_by_email`
filter did not cover it.

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

**Status: ✅ Fixed (2026-08-04)** — the file is now created `0640 root:wiab` with `install`
*before* anything is written, rather than chmod'd afterwards: a later chmod leaves a window, and
the `tee -a` appends from Terraform preserve whatever mode they find. The `wiab` group is
created explicitly rather than relying on `useradd`'s `USERGROUPS_ENAB` default. Takes effect on
re-provision.

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

**Status: ✅ Fixed (2026-08-04)** — `DEFAULT_FORWARD_POLICY` is back to `DROP`, with explicit
rules: established traffic returns; guests are denied the host's primary address, RFC1918,
link-local (cloud metadata), and each other — `172.16.0.0/12` covers the microVM subnet, so
sandboxes are isolated between themselves as well; everything else is allowed out, so agents
keep the internet they need for package installs and API calls.

Scope, stated in the script so the next reader is not misled: these are FORWARD rules, governing
*routed* traffic. A guest reaching the host on its own tap gateway address is INPUT, governed by
ufw's default-deny plus what the firewall section opens — part of why M10 now restricts SSH.

Verified against a real Ubuntu container rather than reasoned about: the `# End required lines`
anchor the insertion depends on exists, and the resulting ruleset is accepted by
`iptables-restore --test`.

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

**Status: ✅ Fixed (2026-08-03)** — four separate parts. Per-IP rate limiting
(`tower_governor`) on login, reset, signup, invite acceptance and token issuance, with a looser
limit for the git transport so a normal clone is not throttled into failing. A semaphore caps
concurrent Argon2 work, bounding peak memory regardless of request rate. Passwords are capped
at 128 characters, enforced in the application services rather than per-handler. Request bodies
are bounded explicitly, with a separate larger limit on the git RPC routes.

The limiter keys on `X-Forwarded-For`, so `frontend/nginx.conf` was changed to *overwrite*
that header rather than append to it — appending leaves a client-chosen value in front and
lets anyone get a fresh bucket per request. The server is also served with connect-info so a
client reaching the backend directly (git over HTTPS) still has an address to key on.

Not done: progressive per-account lockout. Per-IP limiting plus the concurrency cap addresses
the exploitable part; account lockout is itself a denial-of-service vector against a known
username and wants deliberate design.

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

**Status: ✅ Fixed (2026-08-03)** — the login miss paths verify the presented password
against a decoy hash (a real PHC of a value nobody knows, computed once at construction) and
discard the answer, so a failed login costs exactly one verify whether or not the account
exists. Signup spends the same hashing cost on the taken-email branch. Tested by counting
hasher calls rather than timing them. Residual, stated in the code: the signup branches still
differ by a database write and an email send, both far below Argon2.

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

**Status: ✅ Fixed (2026-08-03)** — backslashes are rejected outright rather than reasoned
about, and the destination is re-sanitized at the callback, which is the hop that actually
emits the `Location`.

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

**Status: ✅ Fixed (2026-08-03)** — and it was worse than recorded. The review described a
Firecracker-only path traversal; the Docker runtime interpolates the same unvalidated name into
a container image reference (`<prefix><template>:<tag>`), so `evil.registry.com/x` or
`base@sha256:…` made the backend **pull and run an image of the attacker's choosing**. That is
arbitrary container execution rather than a file read, and it applied to every deployment
rather than only KVM hosts.

`VmTemplate::new` now accepts only `[a-z0-9][a-z0-9._-]*` — an allow-list, in `wiab-core`, so
one choke point covers both sinks and any future runtime inherits it. Reachable from the
`vm_type` field of create-agent and create-team by any member with Write; verified that without
the check, posting `vm_type: "../../etc/passwd"` returns 200 and creates the agent.

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

**Status: ✅ Fixed (2026-08-03)** — 5xx responses now carry a generated reference and
nothing else, with the real message logged at ERROR against the same id. Covers `internal`,
the git `spawn_failure`/`spawn_error` paths, and three raw error responses inside
`authorize_git`. 400s still carry the domain's own validation message, which is the API's
contract. Residual, noted in the code: the application services return `anyhow::Error`, so an
infrastructure failure can in principle reach the 400 path — separating them properly means
typed errors across the application layer.

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

**Status: ✅ Fixed (2026-08-03)** — decompression runs through a size-limited reader, so
memory is bounded during the inflate rather than checked afterwards, and anything past 64 MiB
is refused with 413.

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

### M7 — No HTTP security response headers (frontend, marketing site, and API)

**Status: ✅ Fixed (2026-08-04)** — all three surfaces, each with a policy derived from what it
actually loads rather than a shared template.

- **backend** — `nosniff`, `X-Frame-Options: DENY`, `no-referrer`, and `default-src 'none'`. An
  API serves no scripts or styles of its own, so the useful policy is "nothing". Applied as the
  outermost layer, which a test caught: applied innermost, the headers were missing from the
  401 and 413 that the middleware stack generates before reaching a handler — exactly the
  responses a browser is most likely to be pointed at.
- **frontend** — enforcing CSP with `frame-ancestors 'none'`, plus the other four. This is the
  admin surface, so clickjacking protection is the one that earns its keep. `style-src` keeps
  `'unsafe-inline'` because the app uses React `style={{…}}` in 74 places; `script-src` does not
  need it. Verified by reading the headers off the running container.
- **website** — the CSP ships as **`Content-Security-Policy-Report-Only`**, as this finding
  recommends. It is derived from the real origins (googletagmanager, fonts.googleapis/gstatic,
  the analytics endpoints) but has not been exercised in a browser, and a wrong CSP on the
  marketing site is a blank page. Flip the key once a page load shows no violations. The other
  four headers enforce immediately.

- **Repos:** `frontend`, `website`, `backend`
- **Location:** `frontend/nginx.conf:1-24` (no `add_header` at all);
  `website/firebase.json:11-30` (only `Cache-Control`); backend responses set no
  security headers (`http_api.rs`).
- **Issue:** None of `Content-Security-Policy`, `X-Frame-Options` /
  `frame-ancestors`, `X-Content-Type-Options: nosniff`, `Referrer-Policy`,
  `Permissions-Policy`, or `Strict-Transport-Security` are emitted. The `frontend`
  is an admin surface managing users, tokens, and SSH keys; the
  website loads a third-party GA script and remote fonts with no CSP.
- **Impact:** No clickjacking protection on the frontend (UI-redress against
  destructive actions), no CSP defence-in-depth against any future injection or a
  compromised GA tag, and MIME-sniffing/referrer leakage. (Firebase Hosting does
  emit platform HSTS for the website; the app-level headers are the gap.)
- **Recommendation:** Add a hardened header block on the nginx `**` location and
  a `firebase.json` `headers` entry: `X-Frame-Options: DENY` (or CSP
  `frame-ancestors 'none'`), `nosniff`, `Referrer-Policy:
  strict-origin-when-cross-origin`, a scoped `Permissions-Policy`, and a CSP
  scoped to the actual origins used (self + `googletagmanager.com` /
  `google-analytics.com` / Google Fonts for the website; `default-src 'self'` for
  the frontend). Test with `Content-Security-Policy-Report-Only` first.

### M8 — Website pre-launch gate is client-side only; unreleased content ships to everyone

**Status: 📝 Accepted, as the finding itself recommends (2026-08-03)** — the gate is UX, the
content is marketing copy, and the source already says so. Recorded here so the decision is
explicit rather than inherited. Do not reuse the pattern for anything confidential.

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

**Status: ✅ Fixed (2026-08-04)** — `db_password` has no default and is validated: at least 16
characters from `[A-Za-z0-9._~-]`, a charset that survives being interpolated into a
`DATABASE_URL` and a shell-sourced env file. Rotating the live password is the operator's step.

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

**Status: ✅ Fixed (2026-08-04)** — a new `ssh_admin_cidr` restricts port 22 when set, with a
warning logged at provision time when it is not, and a cloud-init drop-in sets
`PasswordAuthentication no`, `KbdInteractiveAuthentication no` and `PermitRootLogin no`. That
cannot lock anyone out: this host already uses key auth for both the provisioned user and
Terraform's own connection.

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

**Status: ✅ Fixed (2026-08-04)** — the DB password and both Azure SAS URLs are gone from
user-data. Nothing new was built to replace them: the Terraform provisioners already pushed the
same values over SSH on every subsequent deploy, so first boot now uses that path instead of a
second, less safe one. The DB role is created without a password and `provision_db` sets it
afterwards; `wiab-deploy` is deferred to `deploy_app`, which pushes the blob URLs first. Every
provisioner begins with `cloud-init status --wait`, so the deferral is to a step that was
already guaranteed to follow.

This removes the secrets from the rendered cloud-init *inside* Terraform state. It does not
remove the input variables from state — that is C3.

- **Repo:** `iac`
- **Location:** `main.tf:36-60` + `templates/cloud-init.yaml.tftpl:28-35`
  (`WIAB_DB_PASSWORD`, the `WIAB_MODELS_URL`/`WIAB_IMAGES_URL` SAS tokens rendered
  into `provision.env`).
- **Issue:** The DB password and Azure SAS URLs are templated into the VM's
  cloud-init, which is retrievable by anyone with XO/pool access and is stored
  rendered in `terraform.tfstate`.
- **Impact:** A pool operator with XO read access — or anyone with the state file
  (C3) — recovers the DB password and long-lived SAS tokens without touching the
  guest. Note C3 was re-assessed on 2026-08-04 and is not an active leak, so the
  practical path here is XO/pool access rather than the state file.
- **Recommendation:** Avoid durable secrets in user-data; fetch them on the guest
  from a secrets manager at first boot, or inject via the SSH `remote-exec` path.
  Scope SAS tokens tightly (read-only, short TTL, single container).

### M12 — Downloaded binaries and kernels are not checksum- or signature-verified

**Status: ✅ Fixed for the privileged artifacts (2026-08-04)** — firecracker/jailer and
nats-server are now verified against SHA-256 digests pinned in the repository. Pinned there
rather than fetched alongside the artifact: a checksum served by the same host as the file
proves the download was not corrupted, not that it is the file intended.

Verified against the real tarball — the check accepts the genuine file, rejects a corrupted one,
and fails closed on an architecture with no pinned digest.

**Still unverified:** azcopy (fetched via an `aka.ms` redirect) and the smoke-test kernel/rootfs
(operator-supplied URLs, so no digest can be pinned in the repo). Both are lower value than the
VMM — the smoke-test images boot once and are discarded.

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

**Status: ✅ Fixed for the one that matters (2026-08-04)** —
`FirebaseExtended/action-hosting-deploy@v0` was a floating *major* tag on a third-party action
receiving `FIREBASE_SERVICE_ACCOUNT`, in two workflows. Now pinned to the commit `v0` resolved
to on 2026-08-04, so behaviour is unchanged. Note `v0` is *ahead* of the `v0.9.0` release tag,
so pinning the release would have been a silent downgrade.

The remaining unpinned actions are all `actions/*` (GitHub-owned) on major tags. Lower risk;
not done.

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

**Status: ✅ Fixed (2026-08-04)** — the preview workflow is split in two: a `build` job that
runs the branch's code (`npm ci`, `npm run build` — lockfile lifecycle scripts and Vite plugins
from the pull request) and holds no credential, and a `deploy` job that holds the credential and
runs none of the branch's code, receiving `dist` as an artifact. The deploy job does check out
the repo, because the Firebase action reads `firebase.json` for `hosting.public` — config, not
execution. A malicious build step can still tamper with what it produces; it can no longer read
the credential. `release.yml` was already split.

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

**Status: ⚠️ Still open, and the app is now non-functional against the backend (2026-08-03)** —
H2's fix requires the `/signal` socket to authenticate, and the app sends no credentials at
all, so it can no longer join meetings. It already could not list them: it calls
`GET /meetings`, a route that moved under `/organizations/{id}` in the C1 work. This is
knowingly deferred, not overlooked. Doing it properly means OAuth 2.0 authorization-code +
PKCE through the system browser (AppAuth), tokens in Keychain/Keystore, and refresh — and this
backend is an OIDC *relying party*, not a provider, so there is no `/oauth/authorize` for a
native client to use yet. Closing that gap is a feature in its own right and needs its own
design; H9 (cleartext HTTP/ws) must land in the same piece of work, since a bearer token over
`http://` is worthless.

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

**Status: ✅ Fixed (2026-08-03)** — the backend refuses to start with no
`WIAB_GIT_SSH_HOST_KEY` when the advertised base URL is non-local, using the same loopback test
H1 uses for the seeded password. Local development keeps the convenience.

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

**Status: ✅ Fixed (2026-08-03)** — control characters are escaped and the line truncated
before logging, and the reader is bounded so the cap applies before a newline is seen rather
than after.

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

**Status: ✅ Fixed (2026-08-03)** — `UserRepository` gained indexed lookups
(`find_id_by_token_hash` / `find_id_by_ssh_fingerprint`) backed by a new V21 migration. The
indexes are deliberately **not** UNIQUE: nothing rejects a duplicate SSH key today, so an
existing database may hold one and a unique index would fail to create — at boot, since that is
when migrations run. One behaviour change: a shared key now resolves to the lowest user id in
both backends rather than to whichever the scan reached first.

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

**Status: ✅ Fixed (2026-08-03)** — the body is no longer logged; recipient and subject
only.

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

**Status: ✅ Fixed (2026-08-03)** — hand-written redacting `Debug` for both structs, with
tests asserting the secrets never appear in the rendered output.

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

**Status: ✅ Fixed (2026-08-03)** — `validate_password` lives in `authbox-core` and is
enforced by the application services, so a caller that does not go through an HTTP handler is
covered too; the handlers call the same function for an early 400. A maximum length was added
(as a bound on Argon2 work, not a strength rule). The policy immediately caught the seeded
local dev password, which was five characters and would have failed its own rule.

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

**Status: ✅ Fixed (2026-08-03)** — `subtle::ConstantTimeEq`. As the finding says, this was
consistency rather than a live vector: the comparison is over SHA-256 digests, so a timing
oracle would not let anyone construct a matching token.

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

**Status: 📝 Accepted, documented (2026-08-03)** — not fixed, and deliberately so. Turning on
`proxy_ssl_verify` requires nginx to trust the backend's certificate, which the backend
generates for itself at startup unless `WIAB_TLS_CERT`/`KEY` are set. Closing it means giving
both sides a certificate they agree on — a deployment change belonging with the deferred `iac`
work, not a config change. In production this is a `127.0.0.1` hop; in compose it is
container-to-container on a private network. The reasoning is now a comment in `nginx.conf` so
it does not read as an oversight.

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

**Status: ✅ Fixed (2026-08-03)** — both base images pinned by digest, and the serve stage
moved to `nginxinc/nginx-unprivileged`. Verified by running the built image: uid 101, `GET /`
returns 200. It listens on 8080 (a non-root process cannot bind 80), so the compose mapping
moved to `3000:8080`; the host port is unchanged.

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

**Status: ✅ Fixed (2026-08-03)** — `server_tokens off` and a `map $http_upgrade
$connection_upgrade`, bringing the container in line with what production nginx already did.

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

**Status: 📝 Accepted (2026-08-03)** — moving the check server-side needs Firebase custom
claims or a Cloud Function, which is a feature rather than a fix. Accepted for now on the
understanding that the allowlist is **not a secret**: it should not be stored as a GitHub Secret
as though it were, since anyone can read it from the shipped bundle.

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

**Status: ✅ Fixed (2026-08-03)** — a "Cookie settings" link in the footer reopens the banner,
and rejecting now revokes: it denies analytics storage via gtag, stops further events, and
deletes the `_ga` / `_ga_*` / `_gid` cookies against both the exact host and the dot-prefixed
parent domain. The repo had no tests and CI passes with none, so four were added rather than
ship compliance logic unverified.

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

**Status: ✅ Fixed in `backend` (2026-08-03); already fixed in `frontend`/`website`** — the
backend workflow now defaults to `contents: read`, with write granted only to the jobs that
create and upload the release. The build job holds no token at all.

- **Repos:** `backend`, `frontend`, `website`, `dev`, `app`
- **Location:** top-level `permissions: contents: write` in each `release.yml`.
- **Issue:** The write token is granted to every job (build/test/package), not
  just the job that creates the release.
- **Impact:** Any compromised step in a build job (see M13) can push
  commits/tags/releases.
- **Recommendation:** Set the workflow default to `contents: read` and grant
  `contents: write` only on the create-release/upload/deploy job.

### L14 — Tag-derived values interpolated into `run:` shell (latent injection)

**Status: ✅ Fixed in `backend`, `frontend`, `website` (2026-08-03)** — tag and version values
now reach the scripts through `env:`. No `${{ }}` remains inside any `run:` block in those three
repos; the remaining uses are in `env:`/`with:`/`outputs:`, which are not shell contexts. `dev`
still has this pattern.

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

**Status: ✅ Fixed (2026-08-03)** — pinned to `axllent/mailpit:v1.30.6`.

- **Repo:** `backend`
- **Location:** `.github/workflows/ci.yml:95` (`axllent/mailpit`, no tag →
  floating latest; `postgres:16` and the mock OIDC server are pinned).
- **Issue:** The mailpit test-service image is unpinned.
- **Impact:** Low — test-only, no secrets, but non-reproducible and a soft
  supply-chain surface.
- **Recommendation:** Pin `axllent/mailpit@sha256:…` or a version tag.

### L16 — backend/.gitignore has no env/secret patterns

**Status: ✅ Fixed (2026-08-03)** — `.env`, `.env.*` and a `!.env.example` exception added,
matching the other repos.

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

**Status: ✅ Fixed for the password (2026-08-04)** — the DB password is passed to `psql` as a
bound variable (`-v pw=… PASSWORD :'pw'`) instead of being interpolated into the SQL, and M9's
charset validation stops metacharacters reaching the env file or the URL either. The remaining
interpolations carry operator-supplied non-secret values.

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

**Status: ✅ Fixed (2026-08-04)** — the root cause was the backend, not the firewall: the SFU
passed `port_range: None`, so mediasoup allocated anywhere in the ephemeral space and the
firewall had no choice but to open ~50,000 ports. The backend now pins its range
(`WIAB_MEDIASOUP_MIN_PORT`/`MAX_PORT`, default 40000-40999, ~500 concurrent peers) and the ufw
rule opens exactly that — three orders of magnitude smaller. Keep the two in step; a mismatch
shows up as media that negotiates and then carries no audio.

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
- **I8 — Rust RUSTSEC scan now run with `cargo-audit` (`backend`, `dev`);
  supersedes the earlier manual review.** The original review could not run an
  automated scan (the tool was absent) and the manual pin review reported "clean".
  `cargo-audit 0.22.2` has since been installed and run (2026-07-10); it surfaced
  advisories the manual review missed — `backend`: 3 vulnerabilities + 12
  warnings; `dev`: 6 vulnerabilities + 4 warnings. **After reachability triage the
  only reachable, upstream-fixable vulnerability is `dev`'s `rustls-webpki`**;
  `quinn-proto` (both workspaces) is not compiled into the build graph, `rsa`
  (backend) has no patched release, and the `git2`/`rand`/`anyhow` unsoundness
  advisories affect APIs/preconditions that are not exercised. Full per-advisory
  results and remedies are in the "Rust dependency audit (`cargo-audit`)"
  subsection under *Dependency and supply-chain audit*. Still open: wire a
  `rustsec/audit-check` (or `cargo audit --deny warnings`) CI job so new advisories
  are caught automatically going forward.
- **I9 — LICENSE files missing in `docs` and `assets`.** All other repos and the
  org `.github` repo carry MIT. Add for consistency.
- **I10 — No deep links, custom URL schemes, or WebViews in the mobile app
  (`app`).** `MainActivity` is `exported="true"` only for the standard
  `MAIN`/`LAUNCHER` filter (expected). The deep-link and WebView-bridge risk
  classes do not apply; re-audit if schemes/App Links or WebViews are added.

## Dependency and supply-chain audit

| Repo | Tool | Critical | High | Moderate | Low |
|------|------|:--------:|:----:|:--------:|:---:|
| `backend` | `cargo-audit` 0.22.2 | 0 | 1‡ | 2 | 0 |
| `dev` | `cargo-audit` 0.22.2 | 0 | 2‡ | 4 | 0 |
| `app` | `npm audit` | 0 | 0 | 1 | 0 |
| `frontend` | `npm audit` | 1 | 2 | 1 | 0 |
| `website` | `npm audit` | 0 | 0 | 0 | 1 |

Notes:

- `app`/`frontend`/`website` counts include devDependencies. The `website`
  production tree is clean; its one Low is dev tooling.
- The `frontend` critical/high are the vitest and Vite dev-server advisories plus
  the transitive `form-data` — all detailed in M16; the two dev-server issues do
  not affect the shipped static bundle.
- Rust rows count only CVSS/advisory **vulnerabilities** (backend 3, dev 6);
  `cargo-audit` also reported **12 warnings** (backend) / **4 warnings** (dev) for
  unmaintained, unsound, or yanked crates that carry no severity. The full
  per-advisory list with reachability triage and remedies is in the *Rust
  dependency audit (`cargo-audit`)* subsection immediately below.
- **‡ The `High` Rust counts are `quinn-proto`, which is not compiled into either
  build graph** (an optional `reqwest` `http3` dependency that is never enabled) —
  the shipped binaries are not exposed to it. After reachability analysis the only
  reachable, upstream-fixable vulnerability is `dev`'s `rustls-webpki` (the 4
  Moderate in the `dev` row); the backend's 2 Moderate are the `rsa` Marvin
  advisory, which is reachable but has no patched release.
- No dependency is pulled from a git/URL source; no typosquat-looking packages; no
  `overrides`/`resolutions` forcing old versions; all lockfiles present.

### Rust dependency audit (`cargo-audit`) — added 2026-07-10

`cargo-audit 0.22.2` (RustSec advisory DB, 1159 advisories) was run against both
Rust workspaces, superseding the manual pin review referenced in I8. Every
advisory below was triaged for **reachability** — is the crate actually compiled
into the build graph (`cargo tree`), and are the affected APIs called? — because
`cargo-audit` flags every crate present in `Cargo.lock` regardless of whether the
feature that pulls it is enabled. The reachability verdicts were verified, not
assumed:

- **`quinn-proto` is not compiled in either workspace.** It appears in both
  lockfiles only as an optional dependency of `reqwest` behind the `http3`
  feature, which neither crate enables (`reqwest` is pinned
  `default-features = false, features = ["rustls-tls", "json"]`). It is absent
  from `cargo tree -e all` in both workspaces — a lockfile artifact, not a shipped
  code path. This is the single highest-CVSS row in each workspace, and it is not
  exposed.
- **The `git2` unsoundness advisories affect APIs the backend never calls.**
  RUSTSEC-2026-0183 (`Remote::list()`) and RUSTSEC-2026-0184 (a `Signature` from a
  buffer-created `BlameHunk`) — the backend is a git *host* and calls neither
  remotes nor blame (grep for `Remote::list`/`blame`/`BlameHunk`/`find_remote`/
  `.remote(` across `crates/` and `src/` is empty).
- **`rsa` is compiled and reachable, but not via a decryption oracle.** It is
  used for RS256 JWT/JWKS signature *verification* (`openidconnect`, `rsa 0.9.10`)
  and SSH key auth (`russh`/`ssh-key`, `rsa 0.10.0-rc.18`) — public-key operations,
  not the PKCS#1 v1.5 *decryption* of attacker-supplied ciphertext that the Marvin
  timing attack (RUSTSEC-2023-0071) requires. No patched `rsa` exists, so the
  advisory cannot be fully closed; practical exposure here is limited.
- **`rustls-webpki 0.103.9` in `dev` is reachable and trivially fixable.** It does
  real TLS certificate validation via `rustls` ← `reqwest` for the `dev` CLI's
  outbound HTTPS. This is the one clearly-actionable reachable vulnerability, and a
  patch bump clears all four advisories. (The backend's `rustls-webpki` is already
  a fixed version and was not flagged — the `dev` lockfile is simply staler.)
- **`atty` is build-time only** — pulled in as a *build-dependency* of
  `mediasoup-sys` (`planus-translation`), so it never ships in the runtime binary.
  `instant`, `paste`, `audiopus_sys`, and `rustls-pemfile` are transitive deps of
  `mediasoup`/`opus`/`axum-server`; the codebase does not depend on them directly
  and cannot bump them independently of those parents.

#### Vulnerabilities

| Workspace | Crate | Version | Advisory | Sev | Compiled / reachable? | Remedy |
|---|---|---|---|:--:|---|---|
| backend | `quinn-proto` | 0.11.14 | RUSTSEC-2026-0185 (mem exhaustion) | 7.5 High | **No** — `reqwest` `http3` off; absent from build graph | `cargo update -p quinn-proto` (→ ≥0.11.15) to clear the flag; no runtime exposure |
| backend | `rsa` | 0.9.10 | RUSTSEC-2023-0071 (Marvin timing) | 5.9 Med | Yes, via `openidconnect` — signature verify only | No patched release; track upstream, minimize RSA use where an alternative exists |
| backend | `rsa` | 0.10.0-rc.18 | RUSTSEC-2023-0071 (Marvin timing) | 5.9 Med | Yes, via `russh`/`ssh-key` — SSH key auth only | No patched release; track upstream |
| dev | `quinn-proto` | 0.11.13 | RUSTSEC-2026-0185 + RUSTSEC-2026-0037 (DoS) | 8.7 High | **No** — not compiled | `cargo update` (bumps it; not compiled regardless) |
| dev | `rustls-webpki` | 0.103.9 | RUSTSEC-2026-0098/-0099/-0049/-0104 (name-constraint & CRL flaws, CRL-parse panic) | Med | **Yes** — TLS cert validation via `rustls` ← `reqwest` | **`cargo update -p rustls-webpki` (→ ≥0.103.13)** — fixes all four |

**Warnings** (no CVSS; unmaintained / unsound / yanked)

| Workspace | Crate | Version | Advisory | Class | Reachable? | Remedy |
|---|---|---|---|---|---|---|
| backend | `anyhow` | 1.0.102 | RUSTSEC-2026-0190 | unsound (`downcast_mut`) | Yes | `cargo update -p anyhow` → 1.0.103 (fix available) |
| backend | `git2` | 0.20.4 | RUSTSEC-2026-0183 | unsound (`Remote::list` UB) | No — API not called | bump `git2` when a patched release lands |
| backend | `git2` | 0.20.4 | RUSTSEC-2026-0184 | unsound (`BlameHunk` UB) | No — API not called | bump `git2` when patched |
| backend | `rand` | 0.7.3 / 0.9.2 | RUSTSEC-2026-0097 | unsound (custom-logger + `rand::rng()`) | No — precondition not met | `cargo update -p rand` |
| backend | `crypto-bigint` | 0.7.3 | (yanked) | yanked | Yes, via `rsa 0.10`/`crypto-primes`/`elliptic-curve` | `cargo update -p crypto-bigint` off the yanked version |
| backend | `atty` | 0.2.14 | RUSTSEC-2024-0375 / -2021-0145 | unmaintained + unsound | Build-dep only (`mediasoup-sys`) | bump `mediasoup` when available; not in runtime binary |
| backend | `audiopus_sys` | 0.2.2 | RUSTSEC-2026-0150 | unmaintained | Yes, via `opus` (SFU audio) | track/bump `opus`; no direct control |
| backend | `instant` | 0.1.13 | RUSTSEC-2024-0384 | unmaintained | Yes, via `parking_lot` ← `mediasoup` | bump `mediasoup` when available |
| backend | `paste` | 0.1.18 | RUSTSEC-2024-0436 | unmaintained | Transitive | bump parent when available |
| backend | `rustls-pemfile` | 2.2.0 | RUSTSEC-2025-0134 | unmaintained | Yes, via `axum-server` (TLS PEM load) | bump `axum-server` when available |
| dev | `anyhow` | 1.0.102 | RUSTSEC-2026-0190 | unsound (`downcast_mut`) | Yes | `cargo update -p anyhow` → 1.0.103 |
| dev | `lru` | 0.12.5 | RUSTSEC-2026-0002 | unsound (`IterMut` stacked-borrows) | Transitive | `cargo update -p lru` / bump parent |
| dev | `paste` | 1.0.15 | RUSTSEC-2024-0436 | unmaintained | Transitive | bump parent when available |
| dev | `rand` | 0.9.2 | RUSTSEC-2026-0097 | unsound (precondition) | No — precondition not met | `cargo update -p rand` |

#### Recommended actions (in priority order)

1. **Fix the one reachable vulnerability:** in `dev`, `cargo update -p rustls-webpki`
   (moves to ≥0.103.13, clearing RUSTSEC-2026-0098/-0099/-0049/-0104). A broader
   `cargo update` in the `dev` workspace also clears its stale `quinn-proto`.
2. **Clear the trivially-fixable warnings:** `cargo update -p anyhow` (→1.0.103,
   both workspaces) and `cargo update -p crypto-bigint` (off the yanked version),
   then re-run `cargo audit` and confirm the build/tests still pass.
3. **Bump `quinn-proto` in both lockfiles** to silence the highest-CVSS rows even
   though the crate is not compiled — keeps the audit output clean so a real future
   advisory is not lost in noise.
4. **Accept-and-track the residual:** the `rsa` Marvin advisory (no patched crate;
   reachable only via signature verification) and the `mediasoup`/`opus`-chain
   unmaintained crates (`atty`, `audiopus_sys`, `instant`, `paste`,
   `rustls-pemfile`) cannot be fixed from this repo. Record them in a `cargo-audit`
   ignore list (`[advisories] ignore = [...]` in `audit.toml`) **with a written
   justification per entry** so CI stays green while the exceptions remain visible.
5. **Wire `cargo audit` into CI** for both workspaces (see roadmap item 9) so new
   advisories fail the build rather than accumulating silently.

## Secret and credential file inventory

Local files holding real secrets (checked for git-tracking; all correctly
untracked/ignored unless noted):

| File | Repo | Contains | Tracked in git? |
|------|------|----------|-----------------|
| `terraform.tfstate` / `.backup` | `iac` | XO token, DB pw, OIDC/Google secrets, Resend key, SAS URLs (rendered) | No — never in any commit; absent from the project tree as of 2026-08-04, held on the operator's machine only. See C3. |
| `terraform.tfvars` | `iac` | live values of all the above | No — same as above. See C3. |
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
  routes. The frontend stores the session only in the HttpOnly cookie — never
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

1. ~~**Close the API authorization gap (C1, C2).**~~ **Done.** The middleware landed
   2026-07-11; per-resource authorization on the repo browse endpoints landed 2026-08-03,
   along with the integration tests. Note the tests' route table is maintained by hand —
   axum's `Router` does not expose its routes, so a *new* route is not covered
   automatically.
2. ~~**Remediate infrastructure secret handling (C3, H6, M9, M11).**~~ **Done except C3**
   (2026-08-04), which was re-assessed as not an active leak. Original text follows.

   **Remediate infrastructure secret handling (C3, H6, M9, M11).** Move Terraform to an
   encrypted, locked remote backend — for state loss and concurrent-apply corruption as much
   as for confidentiality; make `wiab.env` `0640`; generate a strong DB password; stop
   rendering durable secrets into cloud-init. Blanket rotation is **not** implied: C3 was
   re-assessed on 2026-08-04 and the state was never committed. Rotate if that machine's disk
   has been within reach of a sync, a backup, or a theft.
3. ~~**Fix the boot backdoor and credential logging (H1, L4, L5).**~~ **Done** (2026-08-03).
4. ~~**Authenticate real-time media and harden SSO (H2, H3, H4, M3).**~~ **Done**
   (2026-08-03). Note the media half went further than "bind it to the participant": the
   participant is bound to a user in the aggregate, and the client no longer names one at all.
5. ~~**Harden the hypervisor and sandbox boundary (H5, H7, M12).**~~ **Mostly done**
   (2026-08-04): the microVM FORWARD policy is DROP with scoped rules, and the VMM and broker
   are checksum-verified. Still open: XO TLS verification needs a real certificate on Xen
   Orchestra (the repo default is already `"false"` — check your tfvars), and azcopy plus the
   smoke-test images remain unverified.
6. ~~**Add rate limiting and DoS bounds (M1, M2, M5, M6).**~~ **Done** (2026-08-03), except
   progressive per-account lockout: per-IP limiting plus an Argon2 concurrency cap addresses
   the exploitable part, and account lockout is itself a denial-of-service vector against a
   known username, so it wants deliberate design rather than a default.
7. ~~**Ship security headers and fix the client-side gate (M7, M8).**~~ **Done** (2026-08-04) —
   headers on all three surfaces, with the website's CSP in Report-Only pending a browser
   check. M8 accepted as the finding itself recommends.
8. **Mobile release hardening (H8, H9, M15, M17).** Real release signing key;
   remove blanket cleartext + move to HTTPS/`wss://`; server-side auth for the
   app; enable R8 and stop distributing source maps.
9. ~~**CI/supply-chain hygiene (M13, M14, L13, L14, L15, M16).**~~ **Done** except the breaking
   dependency upgrades (see M16) and SHA-pinning the GitHub-owned `actions/*`. Original text
   follows.

   **CI/supply-chain hygiene (M13, M14, L13, L14, L15, M16).** SHA-pin
   third-party actions; separate untrusted-build from secret-holding deploy jobs;
   scope `contents: write` to the release job; env-indirect tag values; pin the
   mailpit image; run `npm audit`/`cargo audit` in CI and clear the frontend
   advisories. For Rust (see the *Rust dependency audit (`cargo-audit`)*
   subsection): `cargo update -p rustls-webpki` in `dev` (the one reachable vuln),
   `cargo update -p anyhow`/`-p crypto-bigint`/`-p quinn-proto`, and an `audit.toml`
   ignore list with justifications for the residual unfixable advisories (`rsa`
   Marvin, the `mediasoup`/`opus`-chain unmaintained crates).
10. **Remaining low/informational items** as maintenance: constant-time CSRF
    compare, indexed token lookup, persistent SSH host key, backend `.gitignore`
    env patterns, cookie-consent withdrawal, LICENSE files, and the rest.
