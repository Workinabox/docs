# Documentation audit

Date: 2026-08-10

A workspace-wide audit of every documentation artifact against the code as it
actually is, prompted by accumulated drift: architecture docs describing repos
that no longer exist, a product defined three incompatible ways, a security
review whose fixed criticals still read as open, and setup instructions that
point at the wrong files. This document is the report; the corrections it calls
for are tracked in [ROADMAP.md](ROADMAP.md) and the per-repo commits that
followed it.

The workspace is **ten** git repositories: nine siblings — `backend`,
`frontend`, `app`, `website`, `iac`, `dev`, `docs`, `sw-dev-team`, `assets` —
plus `.github` (a real repo holding the org-level architecture docs). The
workspace root folder is **not** a git repository.

## How this was verified

Three exploration passes (frontend inventory; capability-vs-UI mapping;
README/agent-file/security-review audit), then every load-bearing claim below
re-checked against the working tree on 2026-08-10: firewall range vs
`iac/scripts/provision.sh:479` and `backend/crates/wiab-inf/src/sfu.rs:243-244`;
branch example vs `sw-dev-team/src/wiab_team/vcs/worktrees.py:47`;
`LAN_BACKEND_HOST` vs `app/src/backendConfig.ts:5`; crate list vs
`backend/Cargo.toml` members; repo lists vs the actual directories;
`ORG_REPOS` vs `dev/src/org.rs`; tfstate/tfvars existence in `iac/`.

## Per-file verdicts

Legend: ✅ current · ⚠️ stale/incomplete · ❌ wrong (actively misleads) ·
∅ empty · ♻️ duplicate · 🧊 frozen snapshot.

### docs/

| File | Verdict | Note |
|---|---|---|
| AUTH-DOCS.md | ✅ | Matches the implemented auth surface; only gaps are honestly listed as "not done". |
| AGENT_MODEL.md | ⚠️ | Accurate as a design doc, but specs Turn/Step/Stage tracing and security-gate UIs that have no domain code — reads as description, is aspiration. |
| DDD.md | ⚠️ | Methodology sound; opens with "an agentic system for running companies" — one of three conflicting product definitions (see register). |
| SANDBOX_VM.md | ✅ | Firecracker sandbox model matches code; future GUI/computer-use clearly marked as not built. |
| TELEMETRY_FOLLOWUP.md | ✅ | Newest doc (2026-08-10); documents deferred telemetry work. Note: describes what was left out, not how to turn telemetry on. |
| ROADMAP.md | ⚠️ | Mixes done/stale/undecided; #7 (Notion-style PM) overlaps the now-deleted PLANNING_MODEL ontology; no vision statement. Consolidated by this round. |
| SECURITY_REVIEW_OPUS48.md | ⚠️ | Living doc with a good amendment log, but C3's "verified 2026-08-04" block is now false (see register). |
| SECURITY_REVIEW_GPT55.md | 🧊❌ | 23 findings, **zero** status markers; its two Criticals were fixed (they are OPUS48 C1/C2) but nothing says so — every finding reads as open. |
| NOTDONE.md | ⚠️ | The workspace's drift-detector, `Generated 2026-06-09`, itself two months stale — lists shipped systems (git hosting, tasks, teams, PRs, VMs, telemetry) as not-started. Superseded by this document. |
| PLANNING_MODEL.md | ❌ | Drafts a 7-concept ontology and calls the shipped Work/Done model "a throwaway placeholder", with "do not build until we agree" — an undecided fork contradicting shipped code. **Decision: rejected, deleted.** |
| TEMP-PLANNING.md | ♻️ | Raw chat transcript that PLANNING_MODEL.md was distilled from. **Deleted with it.** |
| WEBSITE_REQUIREMENTS.md | ∅ | 188 bytes: header + "overwrite in place" instruction, zero requirements. **Deleted** (website work is parked). |
| CLAUDE.md | ♻️ | One of seven byte-identical copies (2489 bytes). |

### READMEs

| File | Verdict | Note |
|---|---|---|
| website/README.md | ✅ | Best README in the workspace — every version/script verified accurate. |
| iac/README.md | ⚠️❌ | Excellent TLS/hosts discipline, but the WebRTC firewall bullet is wrong (see register) and the Files table omits `images/`, `wiab-cert.sh`, `wiab-deploy.sh`. |
| dev/README.md | ✅ | Accurate for what the tool does — but the tool (`ORG_REPOS`) is stale (see register). |
| dev/local/README.md | ✅ | Thorough, current; the `wiab:wiab` local Postgres default is intentional but unexplained (OPUS48 M9 fix was iac-scoped). |
| backend/README.md | ⚠️ | Overview names 5 capabilities; the service has ~15 subsystems. Env table documents 9 of 78 vars, omitting boot-blocking `WIAB_DEV_OWNER_PASSWORD`. |
| app/README.md | ❌ | Points at `App.tsx` for `LAN_BACKEND_HOST` (wrong file); presents http:// setup as working while the app is non-functional against the backend (see register). |
| sw-dev-team/README.md | ⚠️ | `make`/CI claims verified; but the `work` daemon mode (what production runs) is undocumented, and the layout block omits `worker/`, `api.py`, `telemetry.py`, etc. |
| frontend/README.md | ⚠️ | Two lines; no stack, scripts, or setup. Nothing wrong, everything missing. |
| assets/README.md | ⚠️ | 52 bytes; doesn't link to the real handoff doc at `visual-identity/handoff/README.md`. |
| iac/images/README.md | ❌ | Layout block omits `team/` — the LangGraph runtime's image family that `sw-dev-team` and `backend/agent_runtime.rs` both point at. |
| .github/README.md | ⚠️ | Lists 7 of 10 repos (fixed here in commit 1f08fbb — but the fix missed the sibling architecture docs). |
| .github/profile/README.md | ✅ | The only place the full product thesis is stated — an 8-sentence org blurb. |

### Architecture & agent-guidance files

| File | Verdict | Note |
|---|---|---|
| .github/SYSTEM_ARCHITECTURE.md | ❌ | Documents a `ui/` Leptos/WASM/Trunk repo that does not exist (`SYSTEM_ARCHITECTURE.md:19`); omits 6 of 10 repos. |
| .github/SOFTWARE_ARCHITECTURE.md | ❌ | "`ui` is the browser and desktop-facing Rust/WASM frontend" (`:26`); dependency arrow `wiab → wiab-inf → wiab-core` omits `wiab-app`; names 3 crates, there are 8. |
| .github/AGENTS.md | ❌ | Workspace-layout block lists a `ui/` repo (`:11`) and 4-5 of 10; "do not suggest things unless asked" contradicts the CLAUDE.md posture; points at the stale architecture doc. |
| dev/AGENTS.md | ✅ | The only agent file with real, repo-specific policy ("centre of gravity") — and it is the rule currently being violated (see register). |
| 7× CLAUDE.md | ♻️ | Byte-identical across backend/frontend/app/website/dev/assets/docs; zero repo-specific content; `iac` and `sw-dev-team` have none. |

### sw-dev-team/docs/

| File | Verdict | Note |
|---|---|---|
| ENV.md | ✅ | Verified line by line against `config.py`. |
| ARCHITECTURE.md | ✅ | Matches the graph/state/worktree code. |
| PROTOCOL.md | ❌ | JSON example (`:76`, `:86`) uses `wiab/<run>/dev-1` — the slash form its own §Branch-naming (`:123`) and `worktrees.py:47` say git rejects. |

## Contradiction register

Each entry: what disagrees, and what the code actually does. Disposition in the
final column — **edit** (corrected this round), **decision** (resolved by a
call recorded below), **defer** (left, with where it's tracked).

| # | Contradiction | Code reality | Disposition |
|---|---|---|---|
| 1 | `.github` AGENTS/SYSTEM_ARCH/SOFTWARE_ARCH describe a `ui/` Leptos/WASM repo | No `ui/`, no Leptos; frontend is React 19 + Vite (`frontend/package.json`). Fix landed in `.github/README.md` only (commit 1f08fbb). | edit (Phase C) |
| 2 | Backend is "3 crates" (`SOFTWARE_ARCHITECTURE.md`) | 8 crates in `backend/Cargo.toml`: wiab-{core,inf,app,telemetry,agent}, authbox-{core,app,inf}. | edit (Phase C) |
| 3 | Product = "running companies" (DDD) vs "voice-meeting server" (backend README, SYSTEM_ARCH) vs "vertical work: software/ops/marketing/sales" (website/assets) | All three describe slices of one system; none is complete or current. | decision 1 (describe reality in OVERVIEW.md) |
| 4 | GPT55 presents 23 findings as open; two are Critical | Its C-level findings = OPUS48 C1/C2, fixed 2026-07-11 / 08-03. | edit (Phase D: superseded banner + statuses) |
| 5 | OPUS48 C3 "verified 2026-08-04: no tfstate/tfvars/tfvars exist" | All three exist in `iac/` (68723, 62787, 5119 bytes) since the 2026-08-09 apply — the same doc's own amendment records that deploy. Still git-ignored; still no remote backend. | edit (Phase D: re-verify + amend) |
| 6 | iac/README WebRTC bullet: "unbounded port range", firewall opens `10000-59999/udp` | `provision.sh:479` opens `40000-40999` (~1,000 ports); `sfu.rs:243-244` defaults 40000/40999. provision.sh's own comment already says so. | edit (Phase D) |
| 7 | app/README:22 sets `LAN_BACKEND_HOST` in `App.tsx`; http:// setup shown as working | Symbol is at `src/backendConfig.ts:5`; backend is always-TLS; app broken against it since 08-03 (OPUS48 M15). | edit (Phase D) |
| 8 | PROTOCOL.md JSON uses `wiab/<run>/dev-1` (slash) | `worktrees.py:47` builds `wiab/<run>-dev-<n>` (dash); the doc's own §Branch-naming says the slash form git rejects. | edit (Phase D) |
| 9 | Five different repo lists across `.github`/`dev` docs; `dev/src/org.rs` ORG_REPOS = 7 | Ten repos exist; ORG_REPOS omits docs/iac/sw-dev-team, violating dev/AGENTS.md's own "centre of gravity" rule. | edit (Phase C/D) |
| 10 | `.github/AGENTS.md` "do not suggest things unless asked" vs 7× CLAUDE.md "push back when warranted" | Two active, contradictory instructions to agents. | edit (Phase C: favor CLAUDE.md posture) |
| 11 | ROADMAP #7 "Notion-style PM system" vs PLANNING_MODEL.md ontology | Two competing plans for the same surface; ontology now rejected. | decision 2 |
| 12 | PLANNING_MODEL.md calls shipped Work/Done "a throwaway placeholder" | Work/Done is the shipped, working model. | decision 2 (delete the doc) |
| 13 | web=admin / mobile=meetings split (IDENTITY_AND_ACCESS_PLAN) | Mobile meeting client broken since 08-03; the split's mobile half has no working client. | decision 3 (meetings parked) |
| 14 | Website promises "three security gates" and approve/reject/modify escalations | No gate domain, no approval UI anywhere in code. | decision 4 (keep as roadmap, record as not-built) |
| 15 | Three release schemes (dev release / sw-dev-team tags / website tags); no doc says which is authoritative | All three exist and run independently. | decision 5 (document as-is) |

## Missing documentation

1. Workspace-root README explaining the ten-repo system and bootstrap order.
2. An accurate architecture overview (the `.github` pair documents a system
   that stopped existing ~June).
3. End-to-end getting-started — five docs each hold one fragment (clone,
   `GITHUB_WORKINABOX_TOKEN`, model fetch, image build, sw-dev-team setup) with
   no path between them.
4. A unified release/deploy runbook reconciling the three schemes.
5. A single current-reality product description (this round: `OVERVIEW.md`).
6. A `docs/` index (14 files, no map of living vs frozen).
7. `authbox-*` documentation (three crates, absent from the architecture doc).
8. A telemetry operator doc (how to turn it **on** — currently scattered across
   `config.rs`, `ENV.md`, and TELEMETRY_FOLLOWUP.md).

## Decisions (dispositions of the open forks)

Resolved with the maintainer, 2026-08-10:

1. **Describe reality, not vision.** "Forget the vision for now — make the docs
   consistent with what the system actually is." The canon describes what
   exists; no "companies in a box" framing. Resolves register #3.
2. **PLANNING_MODEL.md and TEMP-PLANNING.md deleted.** The Work/Done model
   shipped is the truth; the ontology is not the direction. Resolves #11, #12.
3. **Meetings parked.** Backend-complete, no working client; not a commitment
   the later frontend work must honor. Resolves #13.
4. **Website promises kept as roadmap.** Copy stays (site is VITE_LAUNCHED-gated,
   pre-launch); docs record gates/approvals/request-access as intended-not-built.
   No website edits this round. Resolves #14.
5. **Release/signup described as-is.** Three release schemes documented as they
   are; invite-only is the working default and self-signup lands role-less —
   documented, not changed. Resolves #15.
