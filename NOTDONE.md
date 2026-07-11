# Not Done

A snapshot of functionality that is **proposed / planned / discussed in the docs but has no implementation in code yet**. Generated 2026-06-09 from an audit of every markdown doc across the repos against the actual code (`backend/`, `frontend/src/`, `website/src/`, `app/`, `dev/src/`).

Two buckets: **(A)** not started at all — nothing in code anywhere; **(B)** started but hollow — a route/page/control exists but renders a placeholder/stub or has no behavior.

When something here gets built, overwrite it in place — the history of what was tackled in what order is part of the document's value.

---

## A. Not started at all

### A1. The agent-execution domain — `docs/AGENT_MODEL.md` is essentially 0% built

Today the backend implements **Meetings** (participants, agenda, floor control, whisper transcription, local llama inference) and **Work** (a hierarchical work-breakdown with acceptance criteria / Done items). None of the agent-execution model exists:

- **Pipeline entities** — `Task` (first-class aggregate), `Turn`, `Step`, `Stage`. No structs/traits/modules. (`Work` is the work-breakdown tree, not the agent `Task`; "AgentTurnSelected" in meetings is floor-control, not a `Turn`.)
- **Board** — the persistent queue where assignable tasks live.
- **Planning / task decomposition** and **reasoning-effort calibration**.
- **Three security gates** — Input Gate, Retrieval Gate, Execution Gate (sender auth, agent scope/role, tool-risk classification).
- **Retrieval / RAG** — dense + sparse + structured retrieval, **query transformation** (HyDE / step-back / sub-question), **reranking**, **retrieval evaluation** (corrective RAG). No embeddings, no vector store, no search.
- **Reasoning stages** — **prompt assembly** (system/user separation), **output parsing** (tool-calls / citations / uncertainty), **reflection / self-critique**, **output guardrails** (claim / citation / policy validation).
- **Agent communication & hierarchy** — delegation / consultation / escalation / notification; agent roles, capabilities, authority levels.
- **Memory systems** — working (context compaction), episodic, semantic (KB), procedural; **memory consolidation** (fact extraction, embedding, audit write).
- **Observability** — structured trace events (Turn → Step → Stage), token/cost tracking, immutable append-only audit log, anomaly detection.
- **Document ingestion & chunking** — per-content-type pipelines, per-chunk metadata / access labels.
- **Prompt-injection defense** and **per-user / per-team / per-agent authorization scoping**.

### A2. Roadmap items with no code — `docs/ROADMAP.md`

Not started:

- **#5** — Workinabox can host git repos
- **#6** — Pipeline/actions system (GitHub Actions–style)
- **#7** — Design a project management system (epics, stories, tasks, Notion-style)
- **#8** — Self-host Workinabox and start using it to build Workinabox
- **#10** — One-command local dev bootstrap (backend + app + dev tooling)
- **#11** — Backend produces a versioned, reproducible container image in CI (no Dockerfile)
- ~~**#12** — Postgres persistence behind the repository traits~~ — **DONE** (2026-06): a `Postgres*Repository` for every aggregate (deadpool/tokio-postgres), chosen by `WIAB_PERSISTENCE` (default `postgres`; `memory` for tests).
- ~~**#13** — Database migration tooling and a migration test in CI~~ — **DONE** (2026-06): refinery tooling (applied on boot; authbox migrations in a separate history table) + the CI `test` job now runs `postgres_integration` against a `postgres:16` service (applies host V1–V11 + authbox V1–V3 migrations on a fresh DB, idempotently, and exercises the repos), plus the Mailpit/OIDC integration tests against service containers.
- **#15** — OpenTelemetry tracing wired inbound→outbound (`tracing` crate present, no OTel export)
- **#16** — Single staging environment reachable from a public URL
- **#17** — Infrastructure-as-code for the staging environment
- **#18** — Secrets management (no secrets in repo, fetched at boot)
- ~~**#19** — Identity provider chosen and integrated for human users~~ — **DONE** (2026-06): in-house `authbox` crate set — local email/password login + sessions, "Continue with Google" and inbound enterprise OIDC/SSO, password reset, admin invite, self-service signup + email verification, deactivate/activate. See `docs/AUTH-DOCS.md`.
- ~~**#20** — Authorization model: roles, scopes, and enforcement middleware~~ — **DONE** (2026-06): `Read/Write/Admin/Owner` over `Org⊇Project⊇Repo`; generic RBAC policy extracted to `authbox-core`; enforced by the per-handler `require_*` guards in `http_api.rs` (session cookie / bearer token / SSH key → principal). See `docs/AUTH-DOCS.md`.
- **#21** — API versioning convention and a contract test harness
- **#22** — In-process domain event bus with a transactional outbox
- **#23** — Background job runner for async work
- **#24** — Error reporting and alerting on staging
- **#25** — Mobile app CI: signed builds to TestFlight and Play internal (`app/` has no CI workflow)
- **#26** — Documentation site published from `docs/`
- **#27** — Production environment promoted from staging, with backup/restore drilled
- **#28** — Feature flags
- **#30** — Task aggregate and board persistence
- **#31** — Retrieval pipeline: ingestion, chunking, vector store, hybrid search
- **#32** — Full inner-loop step with retrieval, tool execution, and reflection
- **#33** — Security gates (Input, Retrieval, Execution) with audit trail
- **#34** — Multi-agent communication (delegation, consultation, escalation, notification)
- **#35** — Memory consolidation across episodic, semantic, and procedural stores
- **#37** — Voice-first UX in the mobile app

Partially started (not in the "not-at-all" set, but incomplete):

- **#14** — Structured logging: `tracing` is used, but correlation IDs are not threaded through every crate.
- **#29** — Thin-slice agent turn: meeting LLM inference exists, but there is no structured Input → LLM → Response turn model.
- **#36** — Meetings feeding the agent model: capture & transcription exist; event extraction and the feedback loop into the agent model do not.

### A3. A doc that is planned but itself empty

- **`docs/WEBSITE_REQUIREMENTS.md`** — header + instruction only; no requirements have been written yet.

---

## B. Started but hollow (placeholder / stub)

### B1. Frontend screens — `frontend/src/App.tsx` → `pages/Placeholder.tsx`

These routes render a generic placeholder: `/board`, `/agents`, `/repos`, `/traces`, `/rooms`, `/security` (Security gates), `/pipelines`.

The one real screen, **Works**, is backed by stub data (`features/works/worksStub.ts`) and has:

- a non-functional filter input (no `onChange` / state / filtering),
- a non-functional "+ New work" button (no `onClick`),
- only 2 of 6 documented status badges (`completed`, `progress`; missing `created`, `assigned`, `blocked`, `failed`).

### B2. Website marketing pages — `website/src/pages/*` → `StubPage`

`/product`, `/the-box`, `/docs`, `/pricing`, `/company` are stubs. Only **Home** is built (Home + i18n + auth modal + cookie/analytics + launch gate are real).

---

## For reference — what *is* built

So this file isn't read as "nothing works": website Home + Firebase hosting + CI; the `dev` CLI (`monitor`, `release`); the backend Meeting + Work aggregates with the repository pattern, SFU/mediasoup signaling, whisper transcription, and local llama inference; git hosting (roadmap #5); and the **authbox identity system** (#19/#20 — local login, Google + enterprise OIDC SSO, sessions, password reset, invites, signup verification, RBAC; `docs/AUTH-DOCS.md`).
