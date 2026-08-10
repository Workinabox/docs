# workinabox docs

Start with **[OVERVIEW.md](OVERVIEW.md)** — what the system actually is today.
Everything else defers to it for current-reality claims.

## Conventions

- **Living** docs are amended in place. When a fact changes, edit it and leave a
  dated note (the style `SECURITY_REVIEW_OPUS48.md` uses — a short amendment line
  — is the model). Don't let a living doc silently drift; that is the failure
  this whole set was just cleaned up from (see `DOCS_AUDIT.md`).
- **Frozen** docs are point-in-time snapshots; they carry a date and are not
  updated (superseded ones get a banner pointing at what replaced them).
- Reality claims belong in `OVERVIEW.md`; forward-looking plans in `ROADMAP.md`.
  A doc that needs to assert "the system does X" should link OVERVIEW rather than
  restate it.

## Index

### Current reality & plan (living)

| Doc | What it is |
|---|---|
| [OVERVIEW.md](OVERVIEW.md) | What workinabox is today: subsystems, the ten repos, surfaces, and what is not built yet. |
| [ROADMAP.md](ROADMAP.md) | Ordered build log. Items marked DONE in place; numbering preserved as history. |
| [DOCS_AUDIT.md](DOCS_AUDIT.md) | The 2026-08-10 doc-vs-code audit: verdicts, contradiction register, resolved decisions. |

### Design & reference (living)

| Doc | What it is |
|---|---|
| [DDD.md](DDD.md) | The domain-driven layering the backend follows (core/app/inf/binary). |
| [AGENT_MODEL.md](AGENT_MODEL.md) | The Task/Turn/Step/Stage agent design. Note: parts (tracing UI, security gates) are design, not yet built. |
| [AUTH-DOCS.md](AUTH-DOCS.md) | The identity/auth system as implemented (authbox). Matches code. |
| [IDENTITY_AND_ACCESS_PLAN.md](IDENTITY_AND_ACCESS_PLAN.md) | Roles/scopes and the intended settings screens. |
| [SANDBOX_VM.md](SANDBOX_VM.md) | Firecracker microVM sandbox model. |
| [TELEMETRY_FOLLOWUP.md](TELEMETRY_FOLLOWUP.md) | Telemetry work deliberately deferred, and how to turn telemetry on. |

### Security reviews

| Doc | What it is |
|---|---|
| [SECURITY_REVIEW_OPUS48.md](SECURITY_REVIEW_OPUS48.md) | Living security review with per-finding status + an amendment log. The current source of truth. |
| [SECURITY_REVIEW_GPT55.md](SECURITY_REVIEW_GPT55.md) | Frozen earlier review, superseded by OPUS48 (see its banner). |

### Archive (frozen / superseded)

| Doc | What it is |
|---|---|
| [archive/NOTDONE.md](archive/NOTDONE.md) | 2026-06-09 doc-vs-code audit, superseded by DOCS_AUDIT.md. |

`CLAUDE.md` in this directory is shared agent-guidance (identical across repos),
not a doc — see `DOCS_AUDIT.md` for the note on those.
