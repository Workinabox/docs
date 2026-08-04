# Roadmap

A walking-skeleton-first ordered list. Each item is the next thing worth doing. Architecture and devops dominate the early items so that everything that comes later (meetings, agent execution, product surface) has solid ground to land on. Existing work is slotted in at the point it makes sense to return to it.

This is a working list. Reorder freely.

When an item is done, overwrite it in place — do not remove it. The numbering and history of what was tackled in what order is part of the document's value.

1. Design and build the public-facing website of workinabox
2. Host the website
3. CI/CD for the website
4. Website reachable at workinabox.ai (canonical apex; workinabox.io redirects to it)
5. Workinabox can host git repos — DONE. Each `Repo` aggregate maps to a bare repo
   under `WIAB_GIT_ROOT`. `git2` powers a REST browse/commit API; real `git
   clone`/`fetch`/`push` is served over Smart-HTTP and SSH by spawning the system
   `git`. Push + the write API are gated by a per-repo push token. (`git_http.rs`,
   `git_ssh.rs`, `git2_backend.rs`; `GitBackend` port in `wiab-core`.)
6. Workinabox has a pipeline/actions system (GitHub Actions–style)
7. Design a Workinabox project management system (epics, stories, tasks, etc., Notion-style) before building it
8. Self-host workinabox and start using it to build workinabox
9. CI runs build, test, fmt, and clippy on every PR across all repos
10. One-command local dev bootstrap (backend + app + dev tooling)
11. Backend produces a versioned, reproducible container image in CI
12. Postgres persistence behind the existing repository traits — DONE. Every aggregate
    has a `Postgres*Repository` behind its trait (deadpool-postgres + tokio-postgres),
    selected at boot by `WIAB_PERSISTENCE` (default `postgres`; `memory` for tests).
    (`pg_pool.rs`, `postgres_*_repository.rs`, `repository_dispatch.rs`.)
13. Database migration tooling and a migration test in CI — DONE. refinery embeds the
    SQL and applies it on boot (`pg_pool::run_migrations`; authbox runs its own series in a
    separate history table). The CI `test` job now provisions a `postgres:16` service and
    runs `postgres_integration` against it — applying **both** series (host V1–V11 + authbox
    V1–V3) idempotently against a fresh DB and exercising the repos — alongside the Mailpit
    and mock-OIDC integration tests against service containers. (`.github/workflows/ci.yml`,
    `wiab-inf/tests/postgres_integration.rs`.)
14. Structured logging with correlation IDs threaded through every crate
15. OpenTelemetry tracing wired from inbound request to outbound call
16. Single staging environment reachable from a public URL
17. Infrastructure-as-code for the staging environment
18. Secrets management (no secrets in repo, fetched at boot)
19. Identity provider chosen and integrated for human users — DONE. Built in-house
    (no external IdP; data in our own Postgres) as the reusable `authbox` crate set
    (`authbox-core`/`-app`/`-inf`): local email/password login (argon2id) with
    server-side browser sessions, "Continue with Google" and inbound **enterprise
    OIDC/SSO** federation (authorization-code + PKCE, JWKS-validated ID tokens),
    forgotten-password reset, admin invite, self-service signup with email verification,
    and user deactivate/activate. Decoupled from WIAB's `User` via a `UserDirectory` port
    so the same crate serves truthdb. Full write-up in `docs/AUTH-DOCS.md`. (`authbox-*`;
    `wiab-inf/http_api.rs` `/auth/*`; `bootstrap.rs` wiring.)
20. Authorization model: roles, scopes, and a middleware that enforces them — DONE.
    `Read < Write < Admin < Owner` roles at `Org ⊇ Project ⊇ Repo` scope; the policy
    (`effective_role`) is extracted into `authbox-core` over abstract
    `ResourceRef`/`ResourceHierarchy`, with WIAB's containment layered on top. Enforced by
    per-handler guards (`require_owner` / `require_org_role` / `require_repo_role` /
    `require_self_or_owner`) that resolve a session cookie **or** bearer token **or** SSH
    key to a principal, with PAT token-scope capping. (`wiab-core/access/`,
    `authbox-core/rbac/`, `authorization_service.rs`, `http_api.rs`.)
21. API versioning convention and a contract test harness
22. In-process domain event bus with a transactional outbox
23. Background job runner for async work
24. Error reporting and alerting on staging
25. Mobile app CI: signed builds distributed to TestFlight and Play internal —
    ANDROID FIRST. CI now runs `test` (Jest) and `build-android` (`assembleDebug`,
    APK uploaded as an artifact) on every PR alongside lint + typecheck; pushing a
    `v*` tag runs `release.yml`, which builds `assembleRelease` and attaches the
    APK to a GitHub Release. (`app/.github/workflows/ci.yml`, `release.yml`;
    `app/jest.config.js`, `jest.setup.js`, `__tests__/App.test.tsx`.)
    Still TODO: (a) the iOS / "mac" side — TestFlight needs macOS runners + Apple
    signing, **deferred until the Android path is solid**; (b) a production keystore
    to replace the debug signing the release APK currently uses, then upload to the
    Play internal track; (c) a real test suite — only one smoke test exists so far,
    many more tests must be added.
26. Documentation site published from `docs/`
27. Production environment promoted from staging, with backup/restore drilled
28. Feature flags
29. First end-to-end thin slice of an agent turn (Input → LLM → Response, no retrieval)
30. Task aggregate and board persistence
31. Retrieval pipeline: ingestion, chunking, vector store, hybrid search
32. Full inner-loop step with retrieval, tool execution, and reflection
33. Security gates (Input, Retrieval, Execution) with audit trail
34. Multi-agent communication (delegation, consultation, escalation, notification)
35. Memory consolidation across episodic, semantic, and procedural stores
36. Return to meetings: capture, transcription, and event extraction feeding the agent model
37. Voice-first UX in the mobile app on top of the meeting and agent stacks
38. ML models must be a hard, fail-fast dependency — not silently disabled. — DONE.
    Both models now load through one symmetric contract: each is independently toggled
    by `WIAB_LLAMA_ENABLED` / `WIAB_WHISPER_ENABLED`, eager-loaded at boot, and HARD-FAILS
    startup when enabled and the model is missing or fails to load (whisper now reports its
    load result over a startup channel, like llama). Disabled means fully off — llama
    disabled ⇒ no meeting intelligence (the `heuristic` fallback and `WIAB_MEETING_INTELLIGENCE`
    are removed; meeting intelligence is now `Option<Arc<dyn MeetingIntelligence>>`).
    Model path vars are filename-only (`WIAB_LLAMA_MODEL_FILE` / `WIAB_WHISPER_MODEL_FILE`),
    resolved against `${WIAB_DATA_DIR}/models/` (default `~/.local/share/wiab` local,
    `/var/lib/wiab` prod — mirrors `WIAB_GIT_ROOT`). (`wiab-inf/src/model_paths.rs`,
    `transcription.rs`, `llama_meeting_intelligence.rs`, `bootstrap.rs::load_meeting_intelligence`,
    `wiab-app/src/meeting_application_service.rs`; `backend/README.md` updated.)
    Model files are fetched (NOT in the app process) from Azure blob storage via `azcopy`,
    using a SAS-bearing `WIAB_MODELS_URL`. Models are treated as a versioned, in-place
    deployable artifact like the binaries: the desired set is a Terraform `models` map keyed
    by UPPERCASE role (`{ LLAMA = { enabled, file }, ... }`), and `wiab-deploy` reconciles it
    on first boot AND every in-place deploy — generically discovering roles from the
    `WIAB_<ROLE>_MODEL_FILE` env, fetching enabled models, syncing the app env, and restarting
    only when the role=file fingerprint changed. Editing the map (add/remove/rename/toggle) is
    the trigger: `terraform apply` SSHes in, refreshes the env, and re-fetches — no version
    knob. Filenames are immutable (new weights ⇒ new filename). Locally:
    `backend/scripts/fetch-models.sh`. (In the iac repo: `scripts/wiab-deploy.sh`, `main.tf`,
    `variables.tf` `models`, `templates/cloud-init.yaml.tftpl`.)
39. Decide on the React Compiler for the frontend. Today it is OFF: `vite.config.ts`
    calls `react()` with no options, and neither `babel-plugin-react-compiler` nor
    `eslint-plugin-react-compiler` is a dependency. Nor is anything done by hand to
    compensate — across 63 files in `frontend/src/` there are exactly three memoization
    calls, all in `features/auth/SessionContext.tsx`. Probably fine for now (small app,
    no measured perf issue), but it should be a decision rather than an omission.
    Suggested order: (a) fix `SessionContext` — it memoizes two callbacks with
    `useCallback` but then passes a fresh object literal as the context value, so every
    `useSession()` consumer re-renders on every provider render and the `useCallback`s
    buy nothing; a `useMemo` on the value object fixes it and is worth doing either way.
    (b) Bump `eslint-plugin-react-hooks` 5 → 6 and turn on its compiler rules — zero
    build cost, and it tells us how compiler-ready the code actually is. (c) Then decide
    on the compiler itself: one line (`react({ plugins: [['babel-plugin-react-compiler',
    {}]] })`) plus a dev dependency, buying automatic memoization, at the cost of an
    extra Babel pass in the otherwise-SWC build — and note the compiler bails out
    silently on components that break the Rules of React, which is why (b) comes first.
40. Decide on dependency update automation across the repos. Today there is none: no
    `dependabot.yml` and no Renovate config in any of the eight repos (docs, backend,
    frontend, app, iac, website, dev, sw-dev-team), so dependencies only move when
    someone remembers to move them. They have drifted accordingly — `frontend/package.json`
    was last touched 2026-06-09 (`513aa2c`), and by 2026-08-04 `npm outdated` showed 18
    packages behind, in three distinct categories worth keeping separate:
    (a) **stale lockfile** — prettier, typescript-eslint, `@types/react(-dom)`, eslint,
    `@testing-library/user-event` are all within the existing semver ranges; `npm update`
    picks them up with no manifest edit. No reason for these to be behind.
    (b) **deliberately pinned exact** — react/react-dom `19.1.1`, vite `7.1.1`,
    `@vitejs/plugin-react-swc` `4.0.0` carry no caret, pinned by `513aa2c` to match the
    website repo. A real choice, not neglect — but the alignment has already drifted
    (website is on `vite: ^7.3.5`). Worth confirming the pinning is still wanted.
    (c) **genuine majors** — typescript 5→7 (the native compiler rewrite), vite 7→8,
    vitest 3→4, eslint 9→10, jsdom 25→29, globals 15→17, jest-dom 6→7,
    react-refresh 0.4→0.5. These need judgment, not a command; being behind on majors
    is often correct.
    The real argument for automation is security patches, not currency: a CVE in a
    transitive dep is the case where "nobody ran the command" gets expensive. The
    counter-argument is PR volume — eight repos spanning npm, Cargo, Terraform, GitHub
    Actions and React Native could mean dozens of PRs a week, and the noise is how these
    get switched off. Dependabot is ~10 lines and native to GitHub but groups poorly;
    Renovate needs more config but can group patches into one weekly PR, auto-merge what
    passes CI, and hold majors for review — the reason to prefer it here. Auto-merge is
    only as trustworthy as the CI behind it: `frontend` has lint + typecheck + test, the
    other repos have not been assessed. Low-risk trial: one repo (`frontend`), grouped
    weekly, auto-merge patch-only, majors held. Roll out if it earns its keep after a
    month; delete one file if it doesn't.
