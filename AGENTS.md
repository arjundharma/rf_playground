<INSTRUCTIONS>
# RF Playground (rf_playground): Multi-Agent RL For Passive IC Layout

## Project Goal And Scope
- Goal: a multi-agent reinforcement learning (MARL) system that collaboratively designs **passive IC components** via layout edits, with simulation-in-the-loop and constraint awareness.
- Initial scope: **passives first** (starting with **spiral inductors** and **MIM/MOM capacitors**), **IC layout**, and an **open PDK** that defines layers, stackup, device rules, and DRC constraints.
- Non-goals (initially): active device sizing/bias, full-chip place/route, signoff-grade DRC/LVS parity with a commercial flow.

## Core Components (System-Level)
These include your original components plus the "glue" needed to make RF/EDA + MARL reproducible and scalable.

### Platform And Architecture
- Modular backend using microservices, containerized with Docker and runnable via `docker compose`.
- API layer(s) for service-to-service coordination and external clients (UI, training jobs, batch optimizations).
- Database for metadata/provenance + an artifact store for large binaries (GDS, S-params, field dumps, plots).
- Job orchestration for simulation/training workloads (queue, retries, timeouts, cancellation, resource limits).
- Observability: structured logs, metrics, traces; cost accounting per sim/episode.

### Technology Choices (Defaults)
- Backend language: Python (pin an explicit version in `pyproject.toml`; keep it consistent across services).
- Python tooling: use `uv` for Python versioning, dependency management, and running tools (no `pip`, no `poetry`).
- Backend web framework: FastAPI (OpenAPI-first APIs; async where it matters).
- Data validation/config: Pydantic + pydantic-settings (typed configs, env-driven).
- DB: Postgres (start here; avoid adding more datastores until needed).
- ORM/migrations: SQLAlchemy + Alembic (schema evolution is mandatory once data exists).
- Auth (initial): basic AuthN/AuthZ scaffolding in the API gateway (JWT + API keys); plan to evolve to OIDC/SSO.
- Observability stack (Docker): Prometheus + Grafana; add Loki (logs), Tempo (traces), and an OpenTelemetry Collector early.
- Frontend: React + TypeScript (Vite recommended for dev speed).
- Testing: pytest (with coverage); tests must run when landing larger functionality changes.

### PDK, Geometry, And Constraints (Critical For IC)
- PDK/version management: layers, vias, units, stackup, device definitions, and rule decks.
- Deterministic geometry generation: pCells must be pure functions of parameters + PDK version.
- Constraint enforcement strategy: "hard filters" (reject invalid actions) vs "soft penalties" (reward shaping).
- DRC/LVS/ERC hooks: start with DRC-like geometric constraints + connectivity sanity checks; evolve to LVS later.

### Layout/CAD Tooling
- Layout generation: `gdsfactory` as the programmatic layout kernel.
- Layout inspection/visualization: KLayout (manual debug + potential rule execution where applicable).
- Modular pCell library for each passive component type (spiral inductors, MIM/MOM caps, resistors, interconnect).

### Simulation Stack (Multi-Fidelity)
- Multi-fidelity evaluation:
  - Fast analytic / quasi-static models for early training signals.
  - Mid-level extraction (RLC / 2.5D approximations) for better accuracy.
  - EM solver integration later (adapter-based; swappable backends).
- Caching/memoization keyed by `(PDK version, layout hash, solver settings, frequency plan)`.
- Simulation promotion logic: decide when to "pay" for higher fidelity based on uncertainty / novelty / promise.

### RL Environment And Multi-Agent Design
- Environment = layout canvas with PDK constraints, ports, and editable regions.
- State representations (choose per experiment): graph/netlist, geometry features, rasterized layout, or hybrids.
- Action space: pCell placement, parameter tweaks, routing edits, topology edits, constraint repair actions.
- Multi-agent coordination: role specialization + shared blackboard state + conflict resolution (locking/regions).
- Reward design: multi-objective (S-params, Q, area, coupling, self-resonance, yield margins) with constraints.

### Interactive UI
- Interactive UI that renders the RL environment and shows edits over time (episode playback).
- Views that matter for debugging: constraint overlays, ports/nets, diffs between revisions, reward breakdowns.
- Human-in-the-loop: override edits, pin constraints, annotate failures, compare Pareto candidates.

### Observability And Logging (Design Requirements)
- Structured logging (JSON) everywhere; include `trace_id`, `span_id`, `request_id`, `layout_revision_id`, `sim_job_id`.
- Prometheus metrics for:
  - Request latency/error rates (per endpoint).
  - Queue depth, job runtimes, retries, timeouts, cancellations.
  - Cache hit rates (by fidelity tier) and artifact store latency.
- Grafana dashboards for: sim throughput/cost, policy eval scores, failure modes (top constraints violated).
- Tracing: instrument services with OpenTelemetry SDKs from day 1; export to an OTel Collector and Tempo.
- Log aggregation: send container/service logs to Loki; correlate logs <-> traces via trace ids in JSON logs.

## Suggested Microservices (MVP -> Scalable)
- `api-gateway`: auth, request routing, experiment/session management, rate limiting.
- `pdk-service`: serves PDK definitions, rule decks, stackups; versioned and immutable.
- `layout-service`: gdsfactory-based pCell instantiation, routing utilities, layout hashing, exporting GDS.
- `rules-service`: constraint checks (DRC-like geometry, connectivity, port validity) and violation reports.
- `sim-service`: simulation adapters, job submission/execution, caching, result normalization, plotting.
- `orchestrator`: workflow engine for "generate -> check -> simulate -> score -> persist".
- `rl-service`: training/inference runners; policy registry; evaluation harness; rollout workers.
- `ui`: web app + backend-for-frontend (if needed) for streaming environment state/events.
- Infra dependencies (Docker Compose): `postgres` (metadata), `prometheus` (metrics), `grafana` (dashboards), `loki` (logs), `tempo` (traces), `otel-collector` (telemetry pipeline), plus optional `redis`/`nats` (queue/events), `minio` (artifacts).

## Data Model (Minimum Viable)
- `PDKVersion`: immutable PDK bundle + semantic version + content hash.
- `PCell`: name, parameters schema, default params, supported layers/ports.
- `LayoutRevision`: `(parent_revision_id, pdk_version_id, layout_hash, gds_uri, metadata_json)`.
- `ConstraintReport`: violations, severities, locations, rule ids; linked to `LayoutRevision`.
- `SimJob`: solver id, settings, frequency plan, resource request, status, logs pointer.
- `SimResult`: normalized metrics (S-params summaries, Q, L/C/R, SRF, coupling), raw artifacts uri.
- `Episode` / `Step`: actions, observations pointers, rewards, done flags.
- `PolicyCheckpoint`: model uri + training config + eval scores.

## Roadmap (Robust Plan For Passive IC MVP)
### Phase 0: Pick A Base PDK And First Component Targets
- PDK choice: use an **open PDK** (to be selected and pinned by exact version/hash).
- First component targets + metrics:
  - Spiral inductor: target `L(f0)`, maximize `Q(f0)`, constrain `SRF > k*f0`, minimize area.
  - MIM/MOM capacitor: target `C`, minimize loss/series R, constrain density/spacing rules.
  - Simple passive networks: L-match / pi / T networks with constraints on insertion loss and matching.

### Phase 1: Deterministic Layout Kernel + pCells
- Implement pCells with strict param schemas and stable hashing.
- Standardize ports/nets: naming, orientation, metal stack usage, and unit conventions.
- Add "validity guards" so illegal geometry is rejected before simulation.

### Phase 2: Constraints First (Before MARL)
- Implement `rules-service` with:
  - Geometry spacing/width/enclosure checks (subset of DRC).
  - Connectivity sanity checks (ports connected, no floating required nets).
- Emit machine-readable violation reports for reward shaping + UI overlays.

### Phase 3: Simulation-in-the-Loop With Multi-Fidelity
- Start with fast models (analytic/quasi-static) to enable many rollouts/day.
- Add extraction fidelity step (RLC / approximations).
- Add EM solver adapter later; gate by promotion logic and caching.

### Phase 4: RL Environment (Single-Agent Baseline)
- Build an environment API that supports:
  - `reset(seed)`, `step(action)`, `render(state)`, `export(layout_revision_id)`.
  - Deterministic replay via stored seeds + action logs.
- Establish baselines: random search, Bayesian optimization, evolutionary strategies, single-agent RL.

### Phase 5: Multi-Agent Collaboration
- Add agent roles + shared blackboard:
  - Placement agent, parameter agent, routing agent, constraint-repair agent, "manager" agent.
- Define edit ownership to prevent thrash: regions, nets, or component instances with locks/leases.
- Add credit assignment mechanisms (per-agent rewards, counterfactual baselines, or shaped auxiliary tasks).

### Phase 6: UI And Human-in-the-Loop
- Episode playback, diffs, constraint overlays, and Pareto explorer.
- "Approve/lock" controls so humans can pin parts of a layout during exploration.

## Engineering Principles (Non-Negotiables)
- Reproducibility: every result ties to `(code rev, PDK hash, pCell params, solver settings, seeds)`.
- Immutability: PDK versions and artifacts are content-addressed; no in-place mutation.
- Adapter interfaces: solvers, rule engines, and PDKs are plug-ins behind stable contracts.
- Safety/sandboxing: sim and plugin execution is isolated; resource-limited; logs captured.

## AuthN/AuthZ Scaffolding (Start Simple, Enable SSO Later)
- Start with: API keys for service-to-service and JWT bearer tokens for user sessions.
- Store: user identities, hashed secrets, API keys, and roles in Postgres (migrated via Alembic).
- Roles (MVP): `admin`, `developer`, `viewer` (expand later); enforce RBAC at the API gateway.
- Plan for SSO: adopt OIDC early in the interface design so we can swap to a provider (e.g., Keycloak/Auth0/Okta)
  without rewriting every service (the gateway validates tokens; internal services trust gateway identity headers).
- Security basics: rotate keys, never log secrets, and isolate "sim execution" from public endpoints.

## Local Config And Secrets (Dev)
- Use a local `.env` for development configuration (compose + services). Do not commit secrets.
- Keep config typed (pydantic-settings) and validated at startup; fail fast on missing required env vars.
- Plan for production secrets later (Vault/SOPS/KMS), but keep the interface env-driven from day 1.

## Networking And Exposure (Safety)
- Default stance: do not expose anything publicly yet.
- Compose networks: keep all services on an internal network; no public ports for DB/queue/artifacts/observability.
- Gateway: the only ingress point by design; when enabled, bind it to localhost only for dev (not `0.0.0.0`).
- Solver/job runners: isolated from ingress; only reachable from internal network and authenticated control plane.

## Artifact Storage (Local First, S3 Later)
- Local dev options to explore:
  - MinIO in Docker Compose (recommended): S3-compatible API backed by a local volume.
  - Filesystem artifact store (fastest to start): content-addressed directory tree under a mounted volume.
- Always access artifacts via a small abstraction (library or `artifact-service`) so the backend can switch to S3
  without changing callers; artifacts should be immutable and addressable by hash/URI.

## Background Jobs (Long Runs, Sim Queues, RL Training)
- Use a background job queue for sim runs and long RL training jobs (HTTP should only submit/control jobs).
- Requirements: retries, timeouts, cancellation, idempotency keys, per-job resource requests, and progress events.
- Local-first choices to evaluate:
  - Redis-backed queue (Celery/RQ/Dramatiq) for simplicity.
  - NATS/JetStream for event-driven workflows if we expect high fan-out.

## Code Practices (Python)
- One `pyproject.toml` per service (or a clear mono-repo layout) with pinned versions; use `uv lock` and commit the lockfile(s).
- Prefer typed code: add type hints on public APIs; use `mypy` or `pyright` (pick one) to prevent drift.
- Use ruff for lint + formatting (single tool reduces CI friction).
- DB hygiene: all schema changes go through Alembic migrations; no ad-hoc SQL in prod paths.
- Determinism: geometry generation and hashing must be stable across machines/containers.
- Contracts: define request/response schemas; never pass around untyped dict blobs between services.

## Testing Practices
- Use pytest for unit + integration tests; add `pytest-cov` for coverage.
- When landing large functionality changes: run tests for affected services (at minimum) before merging.
- Add golden tests for pCells (same params -> same geometry hash) and for constraint checks (known violations).
- Add small end-to-end tests for "generate -> check -> simulate -> score" with the lowest fidelity model.

## CI (Reliable Build From Early On)
- CI should run on every PR:
  - `uv` sync/install, then ruff + typecheck + pytest for backend services.
  - Frontend lint/typecheck/tests (once present).
  - Build Docker images and run a `docker compose` smoke test with healthchecks.
- Add a minimal integration test job that exercises: API -> DB -> queue submission -> result persistence (lowest fidelity).
- Require Docker Compose healthchecks to pass in CI before running integration tests.

## Service Healthchecks (Required)
- Every service in Docker Compose must define a healthcheck (HTTP `/healthz` or equivalent).
- Healthchecks must verify dependencies (e.g., API checks DB connectivity) but be fast and non-flaky.

## Frontend Practices (React + TypeScript)
- Use a typed API client (OpenAPI-generated or thin hand-written client) to avoid schema drift.
- Keep layout rendering decoupled from training backends: UI should be able to replay stored episodes.
- Add basic frontend tests (Vitest for units; Playwright for critical flows) once UI stabilizes.

## Open Questions To Settle Early
- Which base PDK (open vs synthetic) and what rule subset is "enough" for learning signal?
- Which simulation backends are acceptable for on-chip passives (and what fidelity ladder)?
- What frequency plans, normalization, and objective weights define "good" for the first targets?
- How will we handle auth/secrets (local dev vs shared dev) and isolate expensive/unsafe solver execution?
- Which local-first queue and artifact-store choices do we standardize on (so CI/dev match production shape)?
</INSTRUCTIONS>
