# Telemetry follow-ups

What was deliberately left out of the OpenTelemetry instrumentation round
(backend and sw-dev-team branches `otel-instrumentation`, 2026-08-10), in
rough priority order. That round was instrumentation only: everything below
builds on it without touching application code again — the OTLP seam makes
each item a config or deployment change, not a rewrite.

Background: the backend emits JSON logs (trace-correlated) plus an audit
stream on stdout, and exports traces/metrics/logs over OTLP when
`OTEL_EXPORTER_OTLP_ENDPOINT` is set. The team runtime is off by default and
exports when `WIAB_TEAM_OTEL_EXPORTER=otlp`. This closes most of ROADMAP #15
(inbound→outbound tracing); the items here are the remainder.

## 1. iac wiring

Nothing in the deployment sets the new variables yet, so a deployed backend
runs with the zero-config default (JSON logs, no export).

- `OTEL_EXPORTER_OTLP_ENDPOINT` (and optionally `OTEL_SERVICE_NAME`,
  `OTEL_RESOURCE_ATTRIBUTES=deployment.environment.name=…`) into
  `/etc/wiab/wiab.env`, using the established upsert pattern
  (`iac/main.tf` `configure_app`, plus a `triggers` entry and a variable in
  `variables.tf`; `wiab-deploy.sh` `upsert_wiab_env` for deploy-time refresh).
- Team container image: change the install spec in
  `iac/images/team/team-base/Dockerfile` from `[claude,postgres,forge]` to
  `[claude,postgres,forge,otel]`.
- Backend `TeamApplicationService::worker_env` should pass
  `WIAB_TEAM_OTEL_EXPORTER` / `_ENDPOINT` / `_SERVICE_NAME` into launched
  teams (derive from the backend's own OTLP config; the endpoint must be
  reachable from inside the microVM subnet, so it likely needs the gateway
  address, not `127.0.0.1`). `TRACEPARENT` is already injected at the
  VM-runtime layer — do not add it a second time.

## 2. Collector and storage (ClickStack evaluation)

Decided direction: local-first, OTel Collector as the seam, ClickHouse-family
storage; evaluate ClickStack (HyperDX) first, SigNoz as the comparison.

- Dev loop first: a compose service in `dev/local/docker-compose.yml`
  (collector + ClickStack all-in-one) so instrumentation can be inspected
  visually; point the backend at it with `OTEL_EXPORTER_OTLP_ENDPOINT`.
- Then the deployed VM: collector as a systemd service via
  `iac/scripts/provision.sh`, storage sized with a hard disk ceiling
  (ClickHouse TTL, oldest-first drop — ingest must never block).
- Verify licensing per shipped component before bundling anything
  (ClickHouse/OTel Collector are Apache 2.0; check the HyperDX pieces).
- Design the audit stream's path separately: different retention and access
  control from debug telemetry, and a forwarding config (SIEM) that works
  even when nothing else is forwarded.

## 3. Cross-boundary trace verification

Manual check once a collector exists: backend with OTLP set, start a team,
confirm in the UI that
(a) the team's `worker.task` trace links back to the backend's `vm.launch`
trace via the injected `TRACEPARENT`, and
(b) the backend's board-API server spans carry the trace ids of the team's
client spans (`worker/backend.py` injects `traceparent`; `http_trace.rs`
extracts it).

## 4. Frontend and app round

Deliberately skipped this round (user decision). Scope when picked up:
error-boundary reporting and fetch-wrapper timing in `frontend` (the axios
interceptors in `src/auth.ts` are the single seam), `traceparent` on API
requests so backend server spans join the browser's action, and equivalent
minimal instrumentation in `app` around its WebSocket request/response
correlation. The nginx CSP (`connect-src 'self'`) blocks any cross-origin
OTLP endpoint — same-origin collector path or CSP change required. Website
stays GA4-only.

## 5. Outbox and audit durability

- Persist a `traceparent` column in the outbox so NATS publishes carry the
  *originating request's* context rather than the publisher loop's (the
  current caveat is documented in `nats_messaging.rs`).
- A real outbox-lag gauge (needs a `created_at`-based query; today there are
  only structured log events per pass).
- Audit events into the outbox as an `audit.*` aggregate once audit needs
  durable, queryable storage — today the audit stream lives in stdout/OTLP
  logs only, and rotation policy is journald's.

## 6. Team runtime odds and ends

- `WIAB_TEAM_OTEL_CA_PEM` → exporter `certificate_file`, mirroring
  `_trust_backend_certificate`, for an OTLP endpoint behind the backend's
  self-signed TLS.
- Per-checkpoint-write spans via an `AsyncPostgresSaver` wrapper
  (`checkpoint/postgres.py` currently spans setup only).

## 7. Guest agent enrichment

`wiab-agent` stays zero-dependency, but its vsock report lines could carry a
timestamp and the trace id from the `TRACEPARENT` it already receives in
`agent.env`, so the host-side structured `agent report` events correlate.
Also the deferred stretch metric: `wiab.vm.first_report.duration` (launch →
first vsock line), by passing the launch `Instant` into
`spawn_agent_listener`.

## 8. dev repo

Explicitly out of scope: it is a short-lived operator CLI whose `Reporter`
trait is a human-facing progress UI. At most a future `--json` flag on
`PlainReporter`. No OTel.
