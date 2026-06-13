# Plan: Identity, Roles & Access for git hosting

## Context

Git hosting (roadmap #5) is built and working, but its authentication is a placeholder: a
per-repo random token, belonging to no one, jammed into the HTTP Basic password and the
SSH password. That's why normal SSH keys don't work and why everything felt hacky.

The fix is to do auth the way GitHub does it: a **credential** (an access token over
HTTPS, or an SSH public key) identifies a **user**, and the user's **role** decides what
they may do. This requires a real identity model — which the roadmap already anticipated
(#19 identity, #20 authorization). Git hosting simply surfaced the need now.

This plan covers: a `User` aggregate (humans *and* agents), user-owned credentials (SSH
keys + access tokens), an org/project/repo **role** model, an authorization policy, rewiring
both git transports to identity-based auth, **HTTPS/TLS** (replacing plain HTTP), repo
**visibility** for anonymous read, agent identity provisioning, bootstrap seeding, and the
**frontend** settings screens. It removes the per-repo token entirely.

Confirmed decisions: full Read/Write/Admin/Owner roles at Org/Project/Repo scope; repo
Private/Public visibility; agents auto-provisioned a User **with Write on their org**; tokens
`wiab_pat_…`, SHA-256 hashed, shown once, **scopeable** (read-only and/or repo/org-restricted);
TLS folded in; frontend (web console) included; **auth enforced now** on the management API +
console sign-in; **build straight through** all 10 slices, review at the end.

Id convention: aggregate roots (`User` `U-###`, `RoleAssignment` `G-###`) use `*Numbering`
seams; owned entities (`SshKey`, `AccessToken`) use in-domain UUIDs (the `DoneId::new()`
pattern).

## Conventions to follow

Strict hexagonal (`wiab-core` ← `wiab-app` ← `wiab-inf` ← `wiab` binary). Rich domain
models, private fields + fallible constructors, one `thiserror` enum per module,
`*Numbering` seams mint `X-###` ids in inf, `*Snapshot` serde DTOs for HTTP, app services
hold a `Mutex<()>` guard around load-mutate-save, `#[cfg(test)] mod tests` colocated.
Crypto (random token generation, SHA-256 hashing, SSH-key fingerprinting) lives in
inf/app behind seams — **never in core** (core deps stay serde/thiserror/uuid).

---

## 1. Domain (`wiab-core`)

### `user/` — the identity aggregate

- `UserId` (`U-###`).
- `UserKind` — `Human | Agent`.
- `User` aggregate: `id`, `kind`, `name`, `email: Option<String>`, `agent_id:
  Option<AgentId>` (set when `kind == Agent`), `ssh_keys: Vec<SshKey>`, `tokens:
  Vec<AccessToken>`. Credentials are **owned entities** inside the aggregate (they have no
  independent lifecycle). Behaviors: `add_ssh_key`, `remove_ssh_key`, `add_token`,
  `revoke_token`, getters; `snapshot()` (token hashes and key bodies excluded from the
  public snapshot — only labels/displays/fingerprints surface).
- `SshKey` (owned entity): `id` (`K-###`), `label`, `openssh_public_key: String`,
  `fingerprint: String`. Core validates non-empty; the **fingerprint is computed in inf**
  (SHA-256 of the key blob) and passed in.
- `AccessToken` (owned entity): `id` (`T-###`), `label`, `hash: String` (sha256 hex),
  `display: String` (e.g. `wiab_pat_…a1b2`), `created_at`, `expires_at: Option<String>`,
  `last_used_at: Option<String>`, `scope: TokenScope`. The plaintext is **never** stored;
  the hash is computed in app/inf and passed in.
- `TokenScope` value object: `read_only: bool`, `repos: Option<Vec<RepoId>>`, `orgs:
  Option<Vec<OrganizationId>>`. `None` lists = unrestricted on that axis. Method
  `caps(role, target_repo, target_org) -> Option<Role>` to intersect with a computed role.

### `access/` — roles and grants

- `Role` value object: `Read | Write | Admin | Owner` with a total order (`rank()` /
  `PartialOrd`) and `Role::allows(operation: Operation) -> bool`.
- `Operation`: `Read | Write | Administer | Own` (clone/fetch→Read, push→Write, repo
  settings→Administer, org/member management→Own).
- `Scope`: `Org(OrganizationId) | Project(ProjectId) | Repo(RepoId)`.
- `RoleAssignment` aggregate: `id` (`G-###`), `user_id`, `scope`, `role`. Its own aggregate
  so grants are listed/granted/revoked independently.
- **Access policy (pure core fn)**: `effective_role(assignments: &[RoleAssignment], org:
  OrganizationId, project: ProjectId, repo: RepoId) -> Option<Role>` = the max role among
  assignments whose scope covers the repo (its repo, its project, or its org). Keeping the
  rule in core (rich domain); the app layer gathers the data and calls it.

### `repo/` — visibility

- `Visibility` value object: `Private | Public`.
- Add `visibility: Visibility` to `Repo`; `create`/`update` take it (default `Private`);
  `RepoSnapshot` includes it. **Remove** `token`, `authorizes_push`, `rotate_token`, and the
  `RepoToken` module — superseded by user credentials.

---

## 2. Application (`wiab-app`)

- `UserApplicationService` — `create_user`, `user_snapshot`, `list_users`; `add_ssh_key`
  (takes the openssh string; computes fingerprint via an injected `KeyFingerprinter` seam),
  `remove_ssh_key`; `issue_token` (mints plaintext via a `TokenFactory` seam, hashes it,
  stores the `AccessToken`, returns the **plaintext once**), `revoke_token`.
- `AccessApplicationService` — `grant` / `revoke` / list `RoleAssignment`s by user/scope.
- `AuthorizationService` — `effective_role(user_id, repo_id)`: loads the repo → its project
  → the project's org, loads the user's assignments, calls the core `effective_role`
  policy; `authorize(user_id, repo_id, operation, token_scope?) -> bool` applies
  `Role::allows` and intersects with `TokenScope` when a token was used.
- Seams (traits, impls in inf): `TokenFactory` (random `wiab_pat_…` + CRC), `TokenHasher`
  (SHA-256), `KeyFingerprinter` (SHA-256 of key blob). Mirrors the `*Numbering` pattern.
- New request/snapshot types in `*_requests.rs` (create user, add key, issue token with
  scope, grant role, etc.). `IssuedTokenSnapshot { token: TokenSnapshot, plaintext: String }`
  surfaces the secret exactly once.

## 3. Infrastructure (`wiab-inf`)

- In-memory repositories + numbering for `User` and `RoleAssignment` (mirror existing
  `in_memory_*`).
- Seam impls: `Sha256TokenHasher`, `RandomTokenFactory` (uses `rand` + base64url + CRC),
  `Sha256KeyFingerprinter` (parse openssh key, SHA-256 the blob — via `russh::keys`/`ssh-key`).
- **Identity resolution** (`identity.rs`): `resolve_token(plaintext) -> Option<(UserId,
  TokenId, TokenScope)>` (hash → match a stored token, check expiry) and
  `resolve_key(fingerprint) -> Option<UserId>`. In-memory: scan users or maintain
  fingerprint/hash → user indexes updated on save.
- **Rewire `git_http.rs`**: replace the per-repo-token guard with Basic-auth → token →
  `resolve_token` → `AuthorizationService::authorize(user, repo, op, scope)`. Anonymous (no
  creds) allowed only when `repo.visibility == Public && op == Read`. 401 with
  `WWW-Authenticate` otherwise.
- **Rewire `git_ssh.rs`**: **public-key only** (drop password auth and the token field).
  `auth_publickey` → fingerprint → `resolve_key`; accept if found, stash `UserId` on the
  handler; at `exec`, `authorize(user, repo, op)` (clone→Read, push→Write). Keep host-key
  loading/generation. (Anonymous read stays HTTPS-only — matches GitHub.)
- **HTTPS/TLS** (`main.rs`): serve via `axum-server` + `rustls`. Cert/key from `WIAB_TLS_CERT`
  / `WIAB_TLS_KEY` (PEM); else generate a self-signed cert (`rcgen`) for dev with a loud
  warning. Install the rustls ring crypto provider at startup.
- New HTTP routes (in `http_api.rs`), each guarded by token→user→required-role:
  - `POST/GET /users`, `GET /users/{id}`
  - `POST/GET /users/{id}/ssh-keys`, `DELETE /users/{id}/ssh-keys/{key_id}`
  - `POST/GET /users/{id}/tokens` (POST returns plaintext once), `DELETE
    /users/{id}/tokens/{token_id}`
  - `POST/GET /role-assignments` (filterable by user/scope), `DELETE
    /role-assignments/{id}`
  - `PUT /repos/{id}/visibility` (or fold into `update_repo`)

## 4. Bootstrap & agents

- Seed an **Owner** `User` (Human) for the default org with `RoleAssignment Owner @ Org`,
  mint a token, and **log the plaintext on first boot** (dev convenience; replaced by the
  real identity provider, roadmap #19). Resolves the chicken-and-egg of "need a token to
  make a token."
- `AgentApplicationService::create_agent` also **provisions a `User`** (`kind=Agent`,
  `agent_id` linked) and a default `RoleAssignment Write @ Org` so the agent can push to its
  org's repos. The agent gets its own key/token like any user.

## 5. Frontend

There are **two** frontends and they're different surfaces:

- **`app/`** — the React Native mobile app: a meetings/voice app, monolithic `App.tsx`, **no
  navigation framework**, raw `fetch`, `useState`, no management screens.
- **`frontend/`** — the web **console**: React + Vite + Redux Toolkit + axios + react-router,
  with `Sidebar`/`ConsoleLayout` and a repeatable pattern — per entity a
  `features/<x>/{types,api,slice}.ts` plus a `pages/<X>Page.tsx` built on reusable
  `EntityPanel` (list+detail) and `EntityFormModal`. Already has Works/Board/Agents/Repos/
  Pipelines.

Settings (keys, tokens, members, roles) is admin/management UI → it belongs in the **web
console**, which already has the patterns. **The mobile app is out of scope** for settings
(it's a different surface and lacks a nav framework; full role admin on a phone isn't worth
the detour). Repo visibility is a field on the existing console Repos page.

### Console screens (each follows the existing feature/page pattern)

| Area | New page | Sidebar group | Backend |
|---|---|---|---|
| SSH keys | `SshKeysPage` — list (label+fingerprint), add (paste pubkey), delete | Account | `/users/{id}/ssh-keys` |
| Access tokens | `TokensPage` — list (label, `…last4`, created/used/expiry, scope), create → **plaintext shown once** in a modal w/ copy, revoke | Account | `/users/{id}/tokens` |
| Members & roles | `MembersPage` — members of the current org (via existing `ContextSwitcher`); add user, set role at Org/Project/Repo, revoke | Organization | `/role-assignments` |
| Users | `UsersPage` — list/create humans (agents auto-created) | System | `/users` |
| Repo visibility | *no new page* — Private/Public toggle on existing `ReposPage` | — | `PUT /repos/{id}` |

For each: a `features/<x>/{types,api,slice}.ts` (axios + `config.useStub` stub branch like
`reposApi.ts`), the page on `EntityPanel`/`EntityFormModal`, a `Sidebar` nav entry, and a
route in `App.tsx`. Keep stub-DB mode working for the new entities.

### Console must now authenticate

Today the console makes **unauthenticated** axios calls; once the API is protected it must
sign in and send a token. Add: a stored token (localStorage), an **axios interceptor** that
attaches `Authorization`, and a minimal **sign-in (paste-token)** screen seeded by the owner
token printed at boot. Replaced later by the SSO identity provider (roadmap #19).

(The RN mobile app keeps using only the public/meeting endpoints for now; wiring it into the
authed API is a separate, later effort.)

## 6. Token design (decided)

- **Format**: `wiab_pat_` + 32 random bytes (256-bit) base64url + short **CRC** suffix
  (cheap malformed/typo and secret-scanner rejection). One prefix for all; the resolved
  user's `kind` distinguishes human vs agent.
- **Storage**: store **only the SHA-256 hash**; plaintext shown once. SHA-256 (fast,
  deterministic) is correct here — the token is 256-bit random, so a slow KDF (bcrypt/argon2)
  would only add latency to every request for no security gain, and a deterministic hash is
  what enables **O(1) lookup by hash**. Also store a non-secret `display` (prefix+last4).
- **Lifecycle**: optional `expires_at`; `last_used_at` updated on use; revoke = delete.
- **Scope**: `read_only` flag and optional repo/org restriction. Effective permission =
  `min(user_effective_role, token_cap)` — a leaked CI token can be read-only and repo-bound.

## 7. Build order (slices)

1. `User` aggregate + credentials + numbering/repo + `UserApplicationService` + management
   routes + seams (`TokenFactory`/`TokenHasher`/`KeyFingerprinter`).
2. `RoleAssignment` + `Role`/`Scope`/`Operation` + numbering/repo + `AccessApplicationService`
   + routes; core `effective_role` policy.
3. Repo `Visibility` (aggregate + routes); begin removing the per-repo token.
4. `AuthorizationService` + identity resolution (`resolve_token`/`resolve_key`).
5. Rewire `git_http` (token→user→authz, public anon-read) and `git_ssh` (key-only→authz);
   finish removing `RepoToken` and its plumbing.
6. Agent→User provisioning + default org grant.
7. HTTPS/TLS in `main`.
8. Bootstrap seeding (owner user + logged token).
9. Web console: auth (token storage + axios interceptor + sign-in screen), then the
   settings pages (SSH keys, tokens, members/roles, users) + repo visibility toggle, each as
   a `features/<x>` + `pages/<X>Page` following the existing pattern. (Mobile app unchanged.)
10. End-to-end verification.

## 8. Verification

- **Unit (core)**: `User`/credential validation; `Role` ordering + `allows`;
  `effective_role` policy (org/project/repo precedence, max-role); `TokenScope` intersection;
  `Visibility`.
- **Unit (app/inf)**: token issue→hash→resolve round trip; key add→fingerprint→resolve;
  authz decisions across roles/scopes/token caps; in-memory repos.
- **End-to-end (real git, HTTPS + SSH)** against a booted server:
  - create user → add SSH key + issue token → grant role;
  - `git clone`/`push` over **HTTPS** with the token; over **SSH** with the key;
  - **read-only / wrong-scope token** → push denied (403), clone allowed;
  - **Reader role** → push denied, clone allowed; no role → denied;
  - **Public repo** → anonymous HTTPS clone works; SSH still needs a key;
  - agent pushes with its own provisioned identity.
- TLS verified with a real or self-signed cert (`-c http.sslVerify=false` in dev).

## 9. Risks & notes

- Still in-memory (roadmap #12): users/keys/tokens/roles are lost on restart; the seeded
  owner is re-created each boot (with a new logged token). Same caveat as the rest of the
  backend until Postgres lands.
- This is a **breaking change** to git auth: the per-repo token goes away; clients must use a
  user token (HTTPS) or registered key (SSH).
- Self-signed dev TLS → clients must trust the cert or disable verification locally; provide
  a real cert via env for anything shared.
- A lost token can't be recovered (hash-only) — revoke and reissue.
- Agent provisioning must not orphan a `User` if agent creation fails (do it under the same
  guard / roll back on error).
- Frontend is a large slice; backend ships the API first so the app can be built against it.

## Critical files

- New: `wiab-core/src/user/**`, `wiab-core/src/access/**`; `wiab-app/src/user_application_service.rs`,
  `access_application_service.rs`, `authorization_service.rs`, request DTOs;
  `wiab-inf/src/in_memory_user_*.rs`, `in_memory_role_assignment_*.rs`, `identity.rs`,
  token/fingerprint seam impls.
- Edited: `wiab-core/src/repo/{repo.rs,repo_snapshot.rs,mod.rs}` (visibility; remove token);
  `wiab-app/src/repo_application_service.rs` (drop token/verify); `wiab-inf/src/git_http.rs`,
  `git_ssh.rs`, `http_api.rs`, `app_state.rs`; `backend/src/{main.rs,bootstrap.rs}`;
  `backend/Cargo.toml` + `wiab-inf/Cargo.toml` (`sha2`, `axum-server`, `rustls`, `rcgen`);
  the RN app's settings screens + API client.
