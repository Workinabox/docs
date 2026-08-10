# workinabox — what it is today

Date: 2026-08-10 · Living document (amend in place; keep a dated note when a
subsystem's status changes).

This describes the system **as it actually is right now**, not where it is
headed. It exists to replace three conflicting product descriptions scattered
across the repos (see [DOCS_AUDIT.md](DOCS_AUDIT.md)). Forward-looking plans
live in [ROADMAP.md](ROADMAP.md); anything not yet built is listed at the end
here so no reader mistakes intention for reality.

## In one paragraph

workinabox is a self-hosted backend for running agentic software work on your
own hardware or VMs. It hosts identity and access control, git repositories
(over HTTP and SSH), and a project/work/task model; it runs agent **teams** that
claim tasks, do the work inside isolated sandboxes, and open pull requests; and
it carries a voice-**meeting** subsystem in which AI agents participate,
transcribe speech, reply, and produce minutes. The management and CRUD surface
is a React web app; the meeting surface has only a mobile prototype, currently
non-functional. The broader "agentic system that runs whole companies" framing
seen in some marketing and design copy is a direction, not the current product.

## The ten repositories

The workspace is ten git repositories (the root folder itself is not a repo).

| Repo | Language / stack | What it is |
|---|---|---|
| `backend` | Rust (Tokio, axum) | The main service: auth, git hosting, projects/works/tasks/boards, agents/teams, VM runtimes, meetings/SFU, messaging, telemetry. 8 crates. |
| `frontend` | React 19 + Vite + TypeScript | The web management console (see Surfaces). |
| `app` | React Native + TypeScript | Mobile voice-meeting client. Prototype, currently broken against the backend. |
| `website` | React + Vite + Firebase | Public marketing site (`workinabox.ai`), pre-launch. |
| `iac` | Terraform + shell | Provisions the demo host on XCP-ng: VM, TLS, systemd, Firecracker images. |
| `dev` | Rust (clap + ratatui) | Operator CLI for the GitHub org: synchronized release + a monitor dashboard. |
| `sw-dev-team` | Python (LangGraph) | The alternative ("LangGraph") agent runtime; a team of agents in a sandbox. |
| `docs` | Markdown | This documentation set. |
| `assets` | — | Visual identity and design handoff. |
| `.github` | Markdown | Org-level defaults and architecture docs. |

Bootstrap order for a fresh checkout: `backend` (with model files and a local
Postgres — see its README) → `frontend` → optionally `sw-dev-team`, `app`,
`iac`. `dev/local/README.md` gets a full local stack running.

## Backend subsystems and their clients

The backend exposes ~120 HTTP handlers plus a git transport and a WebSocket. Its
crates follow the DDD layering (`*-core` domain, `*-app` use cases, `*-inf`
adapters, `wiab` binary), with a parallel `authbox-{core,app,inf}` trio for the
product-neutral identity kit, plus `wiab-telemetry` and the in-guest
`wiab-agent`.

| Subsystem | State | Web UI | Other client |
|---|---|---|---|
| Auth & identity (local password, Google + enterprise OIDC, PATs, SSH keys, sessions) | ✅ complete | ✅ login/signup/reset/invite/verify + account | — |
| Orgs / projects | ✅ | ✅ context switcher | — |
| Git hosting (smart-HTTP + SSH, branches/files/commits) | ✅ | ⚠️ repo records only — no browsing UI | `git` clients |
| Works / acceptance criteria ("dones") | ✅ | 🟡 list/create; dones read-only | — |
| Boards + tasks (7-state lifecycle, claim/start/block/escalate/complete/fail) | ✅ | ❌ board is a name/description record; no task UI | agent runtimes |
| Agents | ✅ | ✅ create/activate/deactivate | — |
| Teams (create/start/pause/resume/stop) | ✅ | ❌ no UI | — |
| Pull requests (open/list/close/merge) | ✅ | ❌ no UI | — |
| VM sandboxes (Firecracker + Docker runtimes) | ✅ | ❌ internal-only, no HTTP route by design | — |
| Meetings + SFU + STT + agent replies + minutes | ✅ backend | ❌ `/rooms` placeholder | ⚠️ mobile app, broken since 2026-08-03 |
| Agent voice output (TTS) | ❌ stub that errors by design | — | — |
| Messaging (transactional outbox → NATS, 21 domain-event subjects) | ✅ | ❌ nothing subscribes | — |
| Telemetry (OpenTelemetry traces/metrics/logs + audit stream) | ✅ | ❌ no in-product UI | OTLP collector |
| Pipelines | ⚫ name/description shell only (no runs) | ✅ CRUD over an empty domain | — |

Two agent runtimes exist and are selected by `WIAB_AGENT_RUNTIME`: the Rust
`wiab-agent` (the **default**) and the Python `sw-dev-team` LangGraph runtime
(`WIAB_AGENT_RUNTIME=langgraph`).

## Surfaces

- **Web (`frontend`)** — the management/CRUD console: auth, orgs/projects,
  agents, and record-level CRUD for works/boards/repos/pipelines. It does **not**
  yet expose the agent-team runtime (tasks, teams, PRs, code browsing, meetings)
  — roughly the "create a record" third of the API. A rethink toward a
  work-direction + code home is planned (ROADMAP).
- **Mobile (`app`)** — a single-screen React Native voice-meeting client.
  Non-functional against the current backend (TLS + auth + protocol drift; see
  DOCS_AUDIT register #7 and SECURITY_REVIEW_OPUS48 M15). Meetings are parked.
- **CLI (`dev`)** — operator tooling for the GitHub org (release, monitor). Not
  an end-user surface.
- **API** — the real breadth of the product; the widest surface by far, and the
  one the other surfaces lag behind.

## Not built yet

Called out so forward-looking copy elsewhere (notably the marketing site, which
is pre-launch behind `VITE_LAUNCHED`) is not mistaken for shipped behaviour:

- **Security gates** (input / retrieval / execution) — specified in
  AGENT_MODEL.md and promised on the website; no domain code.
- **Human-in-the-loop approvals** (approve / reject / modify on escalation) — no
  approval surface anywhere.
- **Request-access flow** — the website's "Request access" CTA has no backend or
  UI behind it; a fresh self-signup user currently lands with no org role and
  403s on everything.
- **Agent voice output** — TTS is a deliberate stub.
- **A working meeting client** — the subsystem is backend-complete; no client
  works today.
- **Richer work authoring** — works are flat title/description/criteria; a
  proposed richer planning ontology was considered and rejected (the shipped
  Work/Done model stands).
