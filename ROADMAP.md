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
25. Mobile app CI: signed builds distributed to TestFlight and Play internal
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
38. ML models must be a hard, fail-fast dependency — not silently disabled.
    Today the backend silently degrades when model files are absent: meeting
    intelligence falls back to `heuristic` when `WIAB_MEETING_INTELLIGENCE`
    is unset, and transcription returns no-op when `WIAB_WHISPER_MODEL_PATH`
    is unset/missing (`transcription.rs`, `bootstrap.rs::load_meeting_intelligence`).
    The deployed VM runs this way unknowingly. Required:
    - The binary must FAIL TO START if a required model env var is unset or the
      model file is missing (no silent fallback). Remove the silent defaults.
    - The model file is a deployment dependency that travels with the binary
      (fetched at provision/package time to a fixed path, e.g. `/var/lib/wiab/models/`).
    - Fix `backend/README.md` (and `.github/AGENTS.md`) which currently document
      the disable-by-default behavior as if intended.
    - OPEN DECISION (later): where the model files are fetched from. Options:
      Hugging Face direct URL (pin revision + checksum; gemma is gated → needs an
      HF token, TinyLlama/Qwen/whisper-ggml are open), a dedicated GitHub release
      asset used as a permanent bucket (≤2 GB/file), or our own object storage.
