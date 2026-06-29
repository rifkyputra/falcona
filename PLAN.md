# Falcona Implementation Plan

> Turn the [PLATFORM.md](./PLATFORM.md) architecture into a working AI-Native Browser Engineering Platform.

This plan prioritizes a **vertical slice first**, then **breadth**, then ** polish/scale**. The goal is to have a demonstrable end-to-end workflow run within weeks, not months, while leaving room for the ambitious self-improvement and knowledge loops defined in the architecture.

---

## 1. Guiding Principles

1. **Vertical slice over horizontal completeness.** Each milestone must be able to run a real browser workflow end-to-end, even if narrowly scoped.
2. **Agents are first-class packages.** Every agent (Orchestrator, Automation, Investigation, Knowledge) is independently testable and versioned.
3. **Contracts before implementations.** TypeScript interfaces, database schemas, and API contracts are finalized before the code that consumes them.
4. **Observability from day one.** Every step produces structured logs, observations, and artifacts so failures are debuggable.
5. **Human-in-the-loop is a feature, not a fallback.** Build the review queue and confidence scoring early; they shape the agent UX.
6. **Default simple, configurable later.** Hard-coded defaults for thresholds and pools until patterns emerge, then make them tunable.

---

## 2. Milestone Overview

| Milestone | Name | Goal | Approx. Duration | Exit Criteria |
|-----------|------|------|------------------|---------------|
| M0 | Foundation | Monorepo, infra, DB, contracts, CI | 1 week | `pnpm dev` spins up Postgres, Redis, MinIO, API, and UI. Migrations run. |
| M1 | Browser Engine | Libretto integration, browser pool, evidence capture | 1.5 weeks | A script can launch a browser, navigate, click, and store artifacts to S3. |
| M2 | Automation Agent | Step execution, observations, retry budget | 1.5 weeks | API endpoint runs a hard-coded workflow and returns pass/fail with artifacts. |
| M3 | Orchestrator + Planner | Natural-language goal → planned steps → executed run | 2 weeks | POST `/workflows` accepts a goal, plans steps, executes, and stores a run. |
| M4 | Web UI | Dashboard, workflow editor, run inspector | 2 weeks | A user can create a workflow from a prompt and watch it run live. |
| M5 | Investigation Agent | Failure classification, root cause, fix proposals | 2 weeks | A failed run produces an InvestigationReport with artifacts and a suggested fix. |
| M6 | Knowledge Agent + Learning | pgvector, embeddings, selector history, self-improvement | 2 weeks | Similar past runs inform planning; accepted fixes update future confidence scores. |
| M7 | Hardening + Scale | Multi-tenancy, security, rate limits, performance | 2 weeks | Pass load tests; audit log complete; vault integration ready. |

**Total estimated timeline: 12–14 weeks with 2–3 engineers.**

---

## 3. Detailed Milestones

### M0 — Foundation (Week 1)

**Objective:** A runnable development environment and stable contract layer.

#### Tasks
- [ ] Initialize monorepo: `apps/web`, `apps/api`, `packages/*`, `infra/`.
- [ ] Configure pnpm workspaces, Turborepo pipelines, ESLint, Prettier, TypeScript strict mode.
- [ ] Add `docker-compose.yml` with Postgres 16 (pgvector), Redis, MinIO (S3).
- [ ] Set up Drizzle ORM, generate initial migration for all PLATFORM.md tables.
- [ ] Define shared TypeScript contracts in `packages/types`: `Workflow`, `Run`, `Observation`, `InvestigationReport`, `KnowledgeQuery`, etc.
- [ ] Bootstrap Hono API in `apps/api` with `/health` and OpenAPI spec stub.
- [ ] Bootstrap React 19 + TanStack Router in `apps/web` with a basic layout.
- [ ] Add CI workflow (typecheck, lint, test, build) via GitHub Actions.
- [ ] Add `.env.example` and local secret management conventions.

**Deliverables**
- `pnpm dev` starts full stack.
- `/api/health` returns OK.
- UI loads at `http://localhost:3000`.
- Drizzle migrations apply cleanly.

**Decisions to close**
- [ ] Confirm package manager version and Node/Bun runtime choice (recommend Bun for API, Node for UI build tooling).
- [ ] Confirm folder naming convention (`kebab-case` for packages).

---

### M1 — Browser Engine (Week 2–3)

**Objective:** A typed, observable browser execution layer isolated from the AI agents.

#### Tasks
- [ ] Create `packages/browser/action-api` implementing the full `BrowserActionAPI` interface from PLATFORM.md.
- [ ] Integrate Libretto SDK (`launchBrowser`, `instrumentPage`, `extractFromPage`, etc.).
- [ ] Implement `packages/browser/pool`: warm pool with checkout/checkin, TTL, isolation, and crash recovery.
- [ ] Implement `packages/browser/evidence-engine`: capture pre/post DOM snapshots, screenshots, HAR, console logs, and upload bundles to MinIO/S3.
- [ ] Implement `domFingerprint` and `jaccardSimilarity` helpers.
- [ ] Add a manual test script: `pnpm test:browser --url https://example.com` runs navigate + click + screenshot.
- [ ] Add unit tests for fingerprinting and pool state machine.

**Deliverables**
- `BrowserActionAPI` is fully typed and passes a smoke test against a public site.
- Evidence bundles are stored at `s3://artifacts/{runId}/{stepPosition}/`.
- Browser pool recycles sessions and replenishes after crashes.

**Decisions to close**
- [ ] Default browser type for local dev (recommend Chromium).
- [ ] Artifact retention policy for local vs. prod.

---

### M2 — Automation Agent (Week 3–4)

**Objective:** Execute a deterministic workflow step-by-step and capture observations.

#### Tasks
- [ ] Build `packages/agents/automation` as a `pi-agent-core` Agent with browser tools.
- [ ] Implement pre-action routine: pre-observation, fingerprint, drift check against hard-coded baseline.
- [ ] Implement step execution: `navigate`, `click`, `fill`, `extract`, `assert`, `waitForSelector`.
- [ ] Implement retry budget (3 retries) with strategies per failure type from PLATFORM.md.
- [ ] Implement branch handlers for modal, redirect, auth wall, captcha, element missing.
- [ ] Integrate with Run Store: create `Run`, write `Observation` rows, update step status.
- [ ] Create API endpoint `POST /workflows/:id/runs` that accepts a workflow ID and runs it.
- [ ] Seed a hard-coded test workflow in a dev fixture.

**Deliverables**
- API can run a seeded workflow and return `passed`/`failed` with observations.
- Retry budget and branch handling are unit-tested.
- Drift detection flags a changed page.

**Decisions to close**
- [ ] Which branch handlers are real vs. stubbed (e.g., captcha can pause for now).

---

### M3 — Orchestrator + Planner (Week 5–6)

**Objective:** Accept a natural-language goal and produce an executed run via AI planning.

#### Tasks
- [ ] Build `packages/agents/orchestrator`: state machine, retry budget tracking, escalation logic.
- [ ] Implement planner prompt contract (`PlanRequest` → `PlanResponse`) using `pi-ai`.
- [ ] Integrate with `Knowledge Agent` stub for site profiles and prior workflows.
- [ ] Score planned steps using the confidence formula from PLATFORM.md (initially with mock historical data).
- [ ] Route steps below `REVIEW_THRESHOLD` to a pending review state.
- [ ] Wire `POST /workflows` to accept `goal`, plan steps, optionally execute, and return workflow ID.
- [ ] Implement `GET /runs/:id/stream` SSE endpoint for live step progress.
- [ ] Add planner tests with mocked LLM responses.

**Deliverables**
- `POST /workflows` with `{ "goal": "Log in and export CSV" }` plans and runs a workflow.
- Confidence scores are computed and stored per step.
- SSE stream emits step events.

**Decisions to close**
- [ ] Default planner model and provider for `pi-ai`.
- [ ] Whether planning is synchronous or async (recommend async job).

---

### M4 — Web UI (Week 7–8)

**Objective:** A functional React UI for creating, running, inspecting, and reviewing workflows.

#### Tasks
- [ ] Implement Dashboard: active runs, recent failures, site reliability scores (mock data first).
- [ ] Implement Workflow editor: list steps, show confidence badges, reasoning notes.
- [ ] Implement Goal prompt form: user submits a goal, sees planning progress, then step list.
- [ ] Implement Run inspector: step timeline, screenshot viewer, DOM diff/HAR viewer tabs.
- [ ] Implement live run view consuming `/runs/:id/stream`.
- [ ] Implement Review queue: side-by-side expected/observed, approve/reject with selector override.
- [ ] Implement Sites profile page: quirks, selector history, DOM baseline timeline.
- [ ] Add API client (TanStack Query or similar) with typed fetch.

**Deliverables**
- User can create a workflow from a prompt in the UI.
- User can watch a run execute step-by-step.
- User can review a low-confidence step and approve/reject it.

**Decisions to close**
- [ ] UI component library (recommend Tailwind + shadcn/ui or Radix primitives).
- [ ] Screenshot/artifact viewer library.

---

### M5 — Investigation Agent (Week 9–10)

**Objective:** On failure, automatically classify root cause and propose a durable fix.

#### Tasks
- [ ] Build `packages/agents/investigation` as a `pi-agent-core` Agent.
- [ ] Implement artifact bundle loader from Evidence Engine.
- [ ] Compute `DOMDiff` between pre/post accessibility trees.
- [ ] Integrate Libretto `executeRecoveryAgent` / `attemptWithRecovery`.
- [ ] Implement `InvestigationRequest` → `InvestigationReport` prompt.
- [ ] Classify failures into PLATFORM.md categories.
- [ ] Generate `suggestedFix` with replacement steps and confidence.
- [ ] Implement review queue push for `requiresHumanReview`.
- [ ] Implement auto-heal path: create workflow version N+1, apply patch, resume run.
- [ ] Add Investigation Agent tests with synthetic failure bundles.

**Deliverables**
- A failed run triggers an investigation and produces a structured report.
- High-confidence fixes are auto-applied and a new workflow version is created.
- Low-confidence fixes land in the review queue.

**Decisions to close**
- [ ] Auto-heal confidence threshold (start at 0.85, tunable later).
- [ ] How much recovery is delegated to Libretto vs. custom logic.

---

### M6 — Knowledge Agent + Learning (Week 11–12)

**Objective:** The platform learns from every run, improving selectors, baselines, and confidence over time.

#### Tasks
- [ ] Enable pgvector extension and embedding tables in Postgres.
- [ ] Build `packages/agents/knowledge` with vector-search tools.
- [ ] Implement embedding generation for run summaries, failure reports, selector history, DOM fingerprints.
- [ ] Implement `KnowledgeQuery` handlers: `prior_runs`, `selector_alternatives`, `site_quirks`, `dom_baseline`.
- [ ] Integrate Knowledge Agent into Orchestrator planning and Automation Agent pre-action routines.
- [ ] Implement self-improvement ingestion pipeline after resolved failures.
- [ ] Update confidence scoring with real historical data.
- [ ] Add a "Knowledge explorer" UI view.
- [ ] Add tests for vector similarity and self-improvement ingestion.

**Deliverables**
- A selector that failed once and was fixed is preferred on subsequent runs.
- Confidence scores trend upward for stable sites.
- Knowledge explorer surfaces related failures and fixes.

**Decisions to close**
- [ ] Embedding model/provider (recommend small, fast model for embeddings; keep separate from planner model).
- [ ] Vector dimension and index type (e.g., `vector(1536)` + `ivfflat` or `hnsw`).

---

### M7 — Hardening + Scale (Week 13–14)

**Objective:** Production readiness: security, multi-tenancy, rate limits, performance.

#### Tasks
- [ ] Implement `tenant_id` isolation in Run Store (recommend single-schema with `tenant_id` columns for v1).
- [ ] Add audit logging to `audit_events` for every agent decision and human override.
- [ ] Integrate external vault for credentials (HashiCorp Vault or AWS Secrets Manager).
- [ ] Implement per-domain and per-tenant rate limiting.
- [ ] Add sensitive-field masking in observations.
- [ ] Add OpenTelemetry tracing and Prometheus metrics.
- [ ] Add BullMQ job queues for run execution and embedding ingestion.
- [ ] Load-test browser pool and API under concurrent runs.
- [ ] Write runbooks for deployment, monitoring, and incident response.

**Deliverables**
- All actions are audit-logged.
- Credentials are never stored in workflow steps or logs.
- Platform passes load tests for target concurrency.
- Deployment guide and runbooks exist.

**Decisions to close**
- [ ] Finalize multi-tenancy model (single-schema for v1, schema-per-tenant as future option).
- [ ] Hosting and deployment target (Docker Compose for local; Kubernetes or Fly/Railway for prod).

---

## 4. Dependency Graph

```
M0 (Foundation)
  │
  ├─► M1 (Browser Engine)
  │     │
  │     ▼
  ├─► M2 (Automation Agent)
  │     │
  │     ▼
  ├─► M3 (Orchestrator)
  │     │
  │     ▼
  ├─► M4 (Web UI)
  │
  ├─► M5 (Investigation Agent) ── depends on M2, M3
  │     │
  │     ▼
  ├─► M6 (Knowledge Agent) ── depends on M3, M5
  │
  └─► M7 (Hardening) ── depends on all prior
```

**Note:** M4 (UI) and M5/M6 (agents) can proceed in parallel after M3, but the UI should not block agent work.

---

## 5. Open Questions & Decision Log

| # | Question | Current Lean | Owner | Target Resolution |
|---|----------|--------------|-------|-------------------|
| 1 | Multi-tenancy model | Single-schema `tenant_id` for v1 | TBD | M0 |
| 2 | Workflow step DSL | JSON objects (AI-friendly) | TBD | M3 |
| 3 | Replay isolation | Live site by default; HAR replay as debug option | TBD | M5 |
| 4 | Confidence thresholds | Default 0.85 / 0.60, tunable per workflow/site | TBD | M3 |
| 5 | Planner model | Capable reasoning model (e.g., Claude/GPT-4 class) | TBD | M3 |
| 6 | Embedding model | Small/fast model, separate from planner | TBD | M6 |
| 7 | UI component library | Tailwind + Radix/shadcn | TBD | M4 |
| 8 | Deployment target | Docker Compose local; K8s/Fly for prod | TBD | M7 |

Update this log as decisions are made.

---

## 6. Definition of Done (per milestone)

A milestone is done when:

1. All listed tasks are complete and merged to `main`.
2. New code has >70% unit-test coverage for agent and browser packages; UI has smoke tests.
3. API endpoints are documented in the OpenAPI spec.
4. A demo script or UI walkthrough proves the exit criteria.
5. No critical TODOs remain in the changed code.
6. CI passes (typecheck, lint, test, build).

---

## 7. Risk Register

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| Libretto SDK surface changes | Medium | High | Wrap in our `BrowserActionAPI`; minimize direct SDK calls. |
| LLM planner produces invalid step plans | High | Medium | Structured output + Zod validation + retries with error feedback. |
| Browser pool instability under load | Medium | High | Build pool early (M1), load-test in M7, session isolation. |
| Vector search quality is poor initially | Medium | Medium | Start with keyword + semantic hybrid; tune embedding model. |
| Captcha / bot detection blocks runs | High | Medium | Explicit `captcha` branch handler; pause for human; no evasion. |
| Scope creep on UI | Medium | Medium | Strict milestone scope; defer advanced visualizations. |
| Long LLM latency hurts UX | Medium | Medium | Async jobs + SSE; streaming reasoning where supported. |

---

## 8. Recommended First Steps (This Week)

1. **Set up the monorepo** and get `pnpm dev` working with Docker Compose.
2. **Close M0 decisions** around runtime, package manager, and UI stack.
3. **Create the `packages/types` contract file** so downstream work can proceed in parallel.
4. **Write a one-page architecture decision record (ADR)** confirming Libretto integration approach.
5. **Schedule a kickoff review** of this PLAN.md and assign milestone owners.

---

## 9. Appendix: Tech Choices Summary

| Layer | Choice | Rationale |
|-------|--------|-----------|
| Runtime | Bun (API), Node (UI build) | Fast cold starts; broad tool compatibility. |
| Monorepo | Turborepo + pnpm | Caching, workspace isolation, fast installs. |
| API | Hono | Edge-compatible, TypeScript-first, WebSocket support. |
| UI | React 19 + TanStack Router | File-based routing, no framework lock-in. |
| LLM | `@earendil-works/pi-ai` | Unified provider API; model swappable at runtime. |
| Agents | `@earendil-works/pi-agent-core` | Typed tools, event loop, parallel execution. |
| Browser | Libretto SDK + Playwright | Session mgmt, recovery, instrumentation. |
| DB | Postgres 16 + pgvector | Strong transactions + vectors in one store. |
| ORM | Drizzle | Schema-as-code, typed queries. |
| Queue | BullMQ on Redis | Reliable job processing, retries, delayed jobs. |
| Artifacts | S3-compatible (MinIO local / R2 prod) | Cheap, durable object storage. |
| Observability | Pino + OpenTelemetry + Prometheus | Structured logs, traces, metrics. |

---

*Last updated: 2026-06-30*
