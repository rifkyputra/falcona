# AI-Native Browser Engineering Platform

> An autonomous browser engineer that automates workflows, investigates failures, replays and manages actions, analyzes interaction blockers, explains root causes, and continuously improves itself.

---

## Vision

Most browser automation tools wrap Playwright calls in a script runner. This platform is different: an autonomous engineering agent that treats the browser as its domain, not just its tool. The AI layer supplies reasoning, planning, and debugging. Playwright provides the execution engine. Libretto (by Saffron Health) provides the browser toolkit — session management, page instrumentation, network capture, and built-in recovery — sitting between the agent logic and Playwright. Run history, workflow versioning, and replay are handled by a custom Postgres store.

The core bet: browser automation fails not because the scripts are wrong at write time, but because sites change, states are ambiguous, and nothing learns from failures. This platform turns every failure into training data for the next run.

---

## Architecture

```
                    ┌────────────────────────────────────────┐
                    │            Web UI / REST API           │
                    │  React + TanStack Router · Hono · WebSocket push │
                    └──────────────────┬─────────────────────┘
                                       │
                              Goal / Task / Prompt
                                       │
                    ┌──────────────────▼─────────────────────┐
                    │            AI Orchestrator              │
                    │  pi-ai · State machine                  │
                    │  Step planner · Confidence scorer       │
                    │  Human-review gate · Retry budget       │
                    └──────────────────┬─────────────────────┘
                                       │
         ┌─────────────────────────────┼──────────────────────────┐
         ▼                             ▼                          ▼
 Automation Agent            Investigation Agent         Knowledge Agent
 Step planner · Executor     Forensics · Root cause      Memory · Patterns
 Adaptive retry              DOM diff · HAR · Logs       Confidence query
 Branch handler              Selector drift detection    Self-improvement
         │                             │                          │
         ▼                             ▼                          ▼
  Browser Action API           Evidence Engine             pgvector store
  Libretto SDK wrapper         Artifact bundler            Postgres + embeddings
  Observability hooks          Snapshot storage (S3)       Site profiles
         │                             │                          │
         └─────────────────────────────┼──────────────────────────┘
                                       │
              ┌────────────────────────┼──────────────────────┐
              ▼                        ▼                       ▼
       Run Store (Postgres)        Libretto SDK           (shared Postgres)
       Versioned workflows    launchBrowser · workflow     pgvector embeddings
       Run history · Replay   extractFromPage · pageRequest
       Audit log              instrumentPage · executeRecoveryAgent
                                       │
                    ┌──────────────────▼─────────────────────┐
                    │              Playwright                  │
                    │  Browser execution engine               │
                    └──────┬──────────┬──────────┬───────────┘
                           │          │           │
                       Chromium   Firefox     WebKit
```

---

## Tech Stack

| Layer              | Choice                                   | Reason                                              |
|--------------------|------------------------------------------|-----------------------------------------------------|
| API server         | Hono on Bun                              | Edge-compatible, fast cold starts, TypeScript-first |
| Web UI             | React 19 + TanStack Router               | File-based routing, type-safe links, no framework lock-in |
| LLM API            | `@earendil-works/pi-ai`                  | Unified API across 30+ providers (OpenAI, Anthropic, Google, Groq, Mistral, Bedrock…); model is runtime config |
| Agent runtime      | `@earendil-works/pi-agent-core`          | `Agent` class: typed tools, event loop, parallel/sequential tool execution, steer/follow-up |
| Browser toolkit    | `libretto` (saffron-health/libretto)     | `launchBrowser`, `extractFromPage`, `instrumentPage`, `executeRecoveryAgent`, session state |
| Browser execution  | Playwright 1.x (via Libretto)            | Multi-browser, trace viewer, network intercept      |
| Browser pool       | Libretto `BrowserSession` + custom pool  | Session reuse, warm instances, concurrency cap      |
| Job queue          | BullMQ on Redis                          | Priority queues, retries, delayed jobs, visibility  |
| Primary DB         | Postgres 16                              | JSONB for observations, strong transactions         |
| Vector store       | pgvector extension on same Postgres      | Co-located, no separate infra, filterable by site   |
| Artifact storage   | S3-compatible (Cloudflare R2 default)    | HAR traces, screenshots, DOM snapshots              |
| ORM                | Drizzle                                  | Schema-as-code, typed queries, migration control    |
| Realtime push      | WebSocket via Hono upgrade               | Live step status, log streaming to UI               |
| Monorepo           | Turborepo + pnpm workspaces              | Shared types across API / agents / UI packages      |

---

## Data Models

### `workflows`
```sql
CREATE TABLE workflows (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  site_id     UUID REFERENCES sites(id),
  name        TEXT NOT NULL,
  goal        TEXT NOT NULL,          -- original user prompt
  version     INTEGER NOT NULL DEFAULT 1,
  parent_id   UUID REFERENCES workflows(id),  -- lineage for auto-healed versions
  status      TEXT CHECK (status IN ('draft','active','archived')),
  created_at  TIMESTAMPTZ DEFAULT now(),
  updated_at  TIMESTAMPTZ DEFAULT now()
);
```

### `workflow_steps`
```sql
CREATE TABLE workflow_steps (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  workflow_id     UUID REFERENCES workflows(id) ON DELETE CASCADE,
  position        INTEGER NOT NULL,
  action          TEXT NOT NULL,          -- 'navigate' | 'click' | 'fill' | 'extract' | 'assert'
  selector        TEXT,
  value           TEXT,
  options         JSONB DEFAULT '{}',
  confidence      NUMERIC(4,3),           -- 0.000 – 1.000
  requires_review BOOLEAN DEFAULT false,
  notes           TEXT                    -- AI reasoning for this step
);
```

### `runs`
```sql
CREATE TABLE runs (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  workflow_id   UUID REFERENCES workflows(id),
  trigger       TEXT CHECK (trigger IN ('manual','schedule','api','replay')),
  status        TEXT CHECK (status IN ('queued','running','passed','failed','cancelled','needs_review')),
  started_at    TIMESTAMPTZ,
  finished_at   TIMESTAMPTZ,
  error_step_id UUID REFERENCES workflow_steps(id),
  error_summary TEXT,
  meta          JSONB DEFAULT '{}'        -- browser version, viewport, user agent
);
```

### `observations`
```sql
CREATE TABLE observations (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  run_id          UUID REFERENCES runs(id) ON DELETE CASCADE,
  step_id         UUID REFERENCES workflow_steps(id),
  phase           TEXT CHECK (phase IN ('pre','post')),
  url             TEXT,
  dom_fingerprint TEXT,                   -- structural hash for drift detection
  a11y_tree       JSONB,                  -- compressed accessibility tree
  timing_ms       INTEGER,
  artifact_key    TEXT,                   -- S3 key for full artifact bundle
  recorded_at     TIMESTAMPTZ DEFAULT now()
);
```

### `failure_reports`
```sql
CREATE TABLE failure_reports (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  run_id          UUID REFERENCES runs(id),
  step_id         UUID REFERENCES workflow_steps(id),
  category        TEXT CHECK (category IN (
                    'selector_drift','timing_regression','network_error',
                    'unexpected_modal','auth_challenge','captcha','layout_change','unknown'
                  )),
  expected_state  JSONB,
  observed_state  JSONB,
  root_cause      TEXT,                   -- AI-generated explanation
  suggested_fix   JSONB,                  -- proposed corrected step(s)
  resolution      TEXT CHECK (resolution IN ('pending','accepted','rejected','auto_healed')),
  resolved_at     TIMESTAMPTZ
);
```

### `site_profiles`
```sql
CREATE TABLE sites (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  domain          TEXT UNIQUE NOT NULL,
  auth_type       TEXT,                   -- 'basic' | 'oauth' | 'cookie' | 'none'
  known_quirks    JSONB DEFAULT '[]',     -- documented site-specific behaviors
  dom_baseline    JSONB,                  -- last known structural fingerprint per key page
  baseline_run_id UUID REFERENCES runs(id),
  updated_at      TIMESTAMPTZ DEFAULT now()
);
```

---

## Layer Responsibilities

### Web UI / API

**REST API** (Hono router, versioned at `/api/v1`):

```
POST   /workflows                → create workflow from goal prompt
GET    /workflows/:id            → workflow detail with step list
POST   /workflows/:id/runs       → trigger a run
GET    /runs/:id                 → run status + step-level progress
GET    /runs/:id/stream          → SSE stream of live step events
POST   /runs/:id/replay          → replay from a specific step
GET    /runs/:id/artifacts/:step → fetch artifact bundle for a step
GET    /review-queue             → pending low-confidence steps for human approval
POST   /review-queue/:id/approve → approve step and resume run
POST   /review-queue/:id/reject  → reject step with override; AI replans
GET    /sites/:domain/profile    → site profile, quirks, selector history
```

**UI views:**
- **Dashboard**: active runs, recent failures, sites by reliability score
- **Workflow editor**: step list, confidence badge per step, diff view against previous version
- **Run inspector**: step timeline, per-step artifact viewer (screenshot + DOM diff + HAR), Investigation Agent report
- **Review queue**: side-by-side of expected vs. observed state; one-click approve/reject; optional selector override field
- **Knowledge explorer**: search past failures, browse site selector history, view self-improvement timeline

---

### AI Orchestrator

Receives a natural-language goal and runs a planning loop. Implemented as a `pi-agent-core` `Agent` instance backed by `pi-ai` (model configured at startup — e.g. a capable reasoning model for planning, a faster model for scoring). Returns structured output via tool calls.

**Planning prompt contract:**

```typescript
interface PlanRequest {
  goal: string;
  siteProfile: SiteProfile | null;   // fetched from Knowledge Agent
  priorWorkflows: WorkflowSummary[]; // similar past workflows from vector search
  constraints: {
    maxSteps: number;        // default 50
    allowedDomains: string[];
    sensitiveFields: string[]; // fields to mask in logs
  };
}

interface PlanResponse {
  steps: PlannedStep[];
  planConfidence: number;  // 0–1
  warnings: string[];      // e.g. "site uses SPA routing — navigation steps may need waitForURL"
}

interface PlannedStep {
  action: ActionType;
  selector?: string;
  value?: string;
  options?: ActionOptions;
  confidence: number;      // 0–1; see Confidence Scoring
  requiresReview: boolean; // true if confidence < threshold
  reasoning: string;       // one-sentence AI rationale
}
```

**State machine states:**

```
PLANNING → EXECUTING → COMPLETED
                ↓
            INVESTIGATING → AWAITING_REVIEW → REPLANNING → EXECUTING
                                   ↓
                               CANCELLED
```

**Retry budget:** each workflow run starts with a budget of 3 automatic retries per step. The Orchestrator uses this budget before escalating to the Investigation Agent. Retry strategy per failure type:

| Failure type     | Auto-retry strategy                        |
|------------------|--------------------------------------------|
| Timing           | Exponential backoff: 500 ms → 2 s → 8 s   |
| Selector miss    | Try 2 selector fallbacks from Knowledge Agent, then escalate |
| Network error    | Retry immediately up to 2×, then escalate  |
| Modal/overlay    | Dismiss via known pattern, continue; log quirk |
| Auth challenge   | Pull credentials from vault, retry once    |
| Captcha          | Pause, notify human, await override        |

---

### Automation Agent

Implemented as a `pi-agent-core` `Agent` with browser tools registered. Executes the Orchestrator's plan step by step. Never calls Playwright directly — all execution goes through the Browser Action API, which wraps the Libretto SDK.

**Pre-action routine (every step):**

1. Call `getDOMSnapshot()` → store as pre-observation
2. Compute `domFingerprint(snapshot)` → compare against site baseline in Knowledge Agent
3. If drift exceeds threshold (Jaccard similarity < 0.80), flag step for Investigation Agent review before executing
4. Execute action via Browser Action API
5. Call `getDOMSnapshot()` again → store as post-observation
6. Write observation pair to Evidence Engine
7. Write completed step to Libretto

**Branch handling (unexpected state):**

```typescript
type UnexpectedState =
  | { type: 'modal'; selector: string }
  | { type: 'redirect'; to: string }
  | { type: 'auth_wall' }
  | { type: 'captcha' }
  | { type: 'element_missing' }
  | { type: 'unknown' };

// Automation Agent resolution table
const branchHandlers: Record<UnexpectedState['type'], BranchStrategy> = {
  modal:           { action: 'dismiss_and_continue', maxAttempts: 2 },
  redirect:        { action: 'follow_if_in_allowed_domains' },
  auth_wall:       { action: 'inject_credentials_from_vault' },
  captcha:         { action: 'pause_for_human' },
  element_missing: { action: 'try_selector_fallbacks_then_escalate' },
  unknown:         { action: 'escalate_to_investigation' },
};
```

---

### Investigation Agent

Implemented as a `pi-agent-core` `Agent`. Activated by the Orchestrator on failure or significant DOM drift. First allows Libretto's `executeRecoveryAgent` to attempt an immediate in-session fix. If that fails or produces low confidence, it receives the full artifact bundle from the Evidence Engine and produces a structured failure report for human review or auto-heal.

**Investigation prompt contract:**

```typescript
interface InvestigationRequest {
  failedStep: WorkflowStep;
  preObservation: Observation;
  postObservation: Observation | null;  // null if action never completed
  harTrace: HARTrace;
  consoleLogs: ConsoleEntry[];
  domDiff: DOMDiff;                     // structural diff pre vs. post
  priorSuccessfulObservation: Observation | null; // last known-good state for this step
}

interface InvestigationReport {
  category: FailureCategory;
  rootCause: string;            // plain-language explanation
  evidence: {
    selectorDrift?: { was: string; now: string };
    timingDelta?: { expected: number; actual: number };
    networkErrors?: NetworkError[];
    consoleErrors?: string[];
    domChangeSummary?: string;
  };
  suggestedFix: {
    steps: Partial<WorkflowStep>[];  // replacement or patch for failed step(s)
    confidence: number;
    rationale: string;
  };
  requiresHumanReview: boolean;
}
```

**DOM Diff format:**

The Evidence Engine computes a structural diff (not text diff) between pre- and post-action accessibility trees. Elements are identified by their semantic role + text content hash rather than their CSS selector, making the diff stable across selector changes.

```typescript
interface DOMDiff {
  added: DOMNode[];
  removed: DOMNode[];
  modified: { before: DOMNode; after: DOMNode }[];
  selectorMappingChanges: {
    oldSelector: string;
    newSelector: string | null;  // null = element removed entirely
    confidence: number;
  }[];
}
```

---

### Knowledge Agent

Implemented as a `pi-agent-core` `Agent` with vector-search and database tools. Backed by pgvector. Serves two query modes: **similarity search** (planning time) and **exact site lookup** (selector history per domain).

**Stored embedding types:**

| Type               | Content embedded                              | Metadata indexed                         |
|--------------------|-----------------------------------------------|------------------------------------------|
| `run_summary`      | goal text + step descriptions + outcome       | site_id, status, created_at              |
| `failure_report`   | root cause + selector context + page URL      | site_id, category, resolution            |
| `selector_history` | selector string + semantic element description | site_id, page_path, last_success_at     |
| `dom_fingerprint`  | structural tree description                   | site_id, page_path, recorded_at          |

**Query interface:**

```typescript
// Called by Automation Agent before ambiguous actions
interface KnowledgeQuery {
  type: 'prior_runs' | 'selector_alternatives' | 'site_quirks' | 'dom_baseline';
  siteId: string;
  context: string;   // natural language description of current page state / element
  topK: number;      // default 5
}

interface KnowledgeResult {
  matches: {
    content: string;
    similarity: number;
    metadata: Record<string, unknown>;
  }[];
  siteProfile: SiteProfile;
}
```

**Self-improvement ingestion:**

After every resolved `failure_report`, the Knowledge Agent runs:

1. Embed `root_cause + suggested_fix + resolution` → upsert into `failure_report` embedding table
2. Update `selector_history` embedding for the affected selector (old → new mapping)
3. Recompute `dom_fingerprint` baseline for the affected page if the fix was `accepted`
4. Increment `site_profiles.known_quirks` if the failure category was `unexpected_modal` or `captcha`

---

### Browser Action API

The typed isolation boundary between the AI layer and Libretto. Every call delegates to the Libretto SDK (`extractFromPage`, `pageRequest`, Playwright page handles from `launchBrowser`), returns a structured result, and emits an observation record to the Evidence Engine.

```typescript
type Selector = string;

interface ActionOptions {
  timeout?: number;           // ms; default per action type
  force?: boolean;            // bypass actionability checks
  strict?: boolean;           // fail if selector matches >1 element
  retries?: number;           // internal retries before throwing
}

interface ActionResult<T = void> {
  success: boolean;
  value?: T;
  durationMs: number;
  observationId: string;      // reference to the Evidence Engine record
  error?: ActionError;
}

interface ActionError {
  type: 'selector_not_found' | 'timeout' | 'element_not_actionable'
       | 'navigation_failed' | 'network_error' | 'unknown';
  message: string;
  selector?: string;
  pageUrl: string;
}

// Full API surface
interface BrowserActionAPI {
  navigate(url: string, options?: { waitUntil?: 'load' | 'networkidle' }): Promise<ActionResult>;
  click(selector: Selector, options?: ActionOptions): Promise<ActionResult>;
  fill(selector: Selector, value: string, options?: ActionOptions): Promise<ActionResult>;
  select(selector: Selector, value: string, options?: ActionOptions): Promise<ActionResult>;
  check(selector: Selector, options?: ActionOptions): Promise<ActionResult>;
  hover(selector: Selector, options?: ActionOptions): Promise<ActionResult>;
  pressKey(key: string): Promise<ActionResult>;
  extract(selector: Selector, attribute?: string): Promise<ActionResult<string | null>>;
  extractAll(selector: Selector): Promise<ActionResult<string[]>>;
  screenshot(options?: { fullPage?: boolean }): Promise<ActionResult<Buffer>>;
  waitForSelector(selector: Selector, timeout?: number): Promise<ActionResult>;
  waitForNavigation(options?: { url?: string | RegExp }): Promise<ActionResult>;
  getDOMSnapshot(): Promise<ActionResult<DOMSnapshot>>;
  getNetworkLog(): Promise<ActionResult<HARTrace>>;
  getConsoleLog(): Promise<ActionResult<ConsoleEntry[]>>;
  evaluateScript<T>(script: string): Promise<ActionResult<T>>;  // sandboxed; audit-logged
}

interface DOMSnapshot {
  url: string;
  title: string;
  accessibilityTree: A11yNode;
  rawHTML: string;
  fingerprint: string;    // SHA-256 of structural tree (roles + text hashes, no positions)
  timestamp: number;
}
```

---

### Evidence Engine

Captures and stores artifact bundles for every step, every run — not only on failures.

**Artifact bundle structure (stored as JSON + binary in S3):**

```
s3://artifacts/{runId}/{stepPosition}/
  ├── pre.json          // DOMSnapshot (pre-action)
  ├── post.json         // DOMSnapshot (post-action)
  ├── screenshot-pre.png
  ├── screenshot-post.png
  ├── har.json          // full HAR trace for this step's network activity
  ├── console.jsonl     // console entries during this step
  └── diff.json         // DOMDiff between pre and post
```

**Drift detection:**

Fingerprints use Jaccard similarity on the set of `{role}:{textHash}` tokens in the accessibility tree. This is intentionally insensitive to layout, CSS classes, and data attributes — only semantic structure and content matter.

```typescript
function computeFingerprint(a11yTree: A11yNode): string {
  const tokens = extractTokens(a11yTree);  // role:hash pairs, depth-limited to 4
  return sha256(tokens.sort().join('|'));
}

function jaccardSimilarity(a: string[], b: string[]): number {
  const setA = new Set(a), setB = new Set(b);
  const intersection = [...setA].filter(x => setB.has(x)).length;
  const union = new Set([...setA, ...setB]).size;
  return intersection / union;
}

const DRIFT_THRESHOLD = 0.80;  // below this → alert before executing action
```

---

### Confidence Scoring

Every planned step receives a composite confidence score before it is executed or queued for human review.

**Scoring factors:**

| Factor                        | Weight | Source                                          |
|-------------------------------|--------|-------------------------------------------------|
| Selector historical success   | 0.30   | `selector_history` table: successes / total attempts |
| DOM similarity to prior runs  | 0.25   | Jaccard similarity of current fingerprint vs. baseline |
| Action type risk              | 0.20   | Static table: `navigate`=0.95, `click`=0.85, `fill`=0.80, `evaluateScript`=0.50 |
| Site familiarity              | 0.15   | Number of past successful runs for this domain (log-scaled, saturates at 50 runs) |
| AI planner stated confidence  | 0.10   | Raw value from planning LLM                     |

```typescript
function scoreStep(factors: ConfidenceFactors): number {
  return (
    factors.selectorSuccessRate    * 0.30 +
    factors.domSimilarity          * 0.25 +
    factors.actionTypeBaseline     * 0.20 +
    factors.siteFamiliarity        * 0.15 +
    factors.plannerConfidence      * 0.10
  );
}

// Routing thresholds
const AUTONOMOUS_THRESHOLD = 0.85;  // run without human review
const REVIEW_THRESHOLD     = 0.60;  // queue for human review
// below 0.60 → block execution, require human override
```

---

### Libretto SDK

Third-party browser toolkit (`saffron-health/libretto`) that sits between the Browser Action API and Playwright. The platform uses it as an SDK, not a CLI.

**Key SDK exports used by this platform:**

```typescript
import {
  launchBrowser,        // BrowserSession — manages the Playwright browser/page lifecycle
  workflow,             // define a replayable workflow (stores state in .libretto/)
  extractFromPage,      // AI-powered semantic extraction using accessibility tree
  pageRequest,          // make typed HTTP requests via the browser's network context
  instrumentPage,       // inject ghost cursor, highlights, action recording
  executeRecoveryAgent, // built-in AI recovery loop when a step fails
  attemptWithRecovery,  // wrap any action with automatic recovery on failure
  downloadViaClick,     // handle file downloads triggered by clicking
  pause,                // pause execution for debugging
} from 'libretto';
```

**`executeRecoveryAgent` / `attemptWithRecovery`:**
Libretto has a built-in recovery agent (`COMPUTER_USE_RECOVERY_MODELS`) that activates on failure, inspects the current page state, and attempts corrective actions. The Investigation Agent consumes Libretto's recovery output as evidence, adds root cause reasoning on top, and proposes a durable fix for the workflow record.

**Session state:**
Libretto persists session state (auth profiles, browser cookies, recording data) in a `.libretto/` directory. The platform treats this as ephemeral browser state — durable workflow records live in the Run Store (Postgres).

---

### Run Store

Custom Postgres-backed component (Drizzle ORM). Agents write to it; the UI reads from it. It does not direct agent behavior.

**Replay semantics:**

A replay creates a new `run` row linked to the original `workflow_id`. The caller specifies a `fromStepPosition` (0-indexed). The replay run re-executes from that step forward using the Libretto SDK, loading observations from the original run as context for the AI layer.

**Versioning:**

When the Investigation Agent proposes a fix that modifies one or more `workflow_steps`:

1. Copy current workflow as `version N+1` with `parent_id` pointing to version N
2. Apply proposed step patches to the new version
3. Mark old version `archived` if the fix is accepted
4. Maintain the full lineage chain so any prior version can be restored

---

## Confidence Scoring Worked Example

Goal: `"Log into the admin dashboard and export the user CSV"`

| Step | Action         | Selector                        | Score | Routed to        |
|------|----------------|---------------------------------|-------|------------------|
| 1    | `navigate`     | `https://app.example.com/login` | 0.97  | Autonomous       |
| 2    | `fill`         | `#email`                        | 0.91  | Autonomous       |
| 3    | `fill`         | `#password`                     | 0.86  | Autonomous       |
| 4    | `click`        | `button[type=submit]`           | 0.89  | Autonomous       |
| 5    | `waitForURL`   | `/dashboard`                    | 0.92  | Autonomous       |
| 6    | `click`        | `nav >> text="Users"`           | 0.72  | Autonomous       |
| 7    | `click`        | `button >> text="Export"`       | 0.61  | Autonomous (low) |
| 8    | `click`        | `.export-modal >> text="CSV"`   | 0.54  | **Human review** |
| 9    | `waitForDownload` | —                            | 0.88  | Autonomous       |

Step 8 is queued for human review because the modal selector has never been seen before (`selector_history` = 0 runs) and the DOM fingerprint for the export modal page does not match any baseline (site familiarity factor is penalized). The user sees a side-by-side screenshot, approves the selector, and the run continues autonomously.

---

## Data Flows

### Happy Path

```
User submits goal
  → Orchestrator: embed goal → vector search for similar prior workflows
  → Orchestrator: fetch site profile for target domain
  → Orchestrator (pi-agent-core Agent backed by pi-ai): plan N steps with confidence scores
  → Steps scored < REVIEW_THRESHOLD → queued for human review (non-blocking if 0 such steps)
  → For each step:
      Automation Agent:
        ① getDOMSnapshot() → pre-observation → fingerprint
        ② compare fingerprint vs. Knowledge Agent baseline
        ③ if drift < threshold: execute via Browser Action API
        ④ getDOMSnapshot() → post-observation
        ⑤ Evidence Engine: bundle pre/post/HAR/console → S3
        ⑥ Run Store: mark step complete, link observation IDs
  → All steps complete → run status = 'passed'
  → Knowledge Agent: ingest run summary embedding; update site familiarity counter
```

### Failure Path

```
Automation Agent: step N action fails (timeout / selector miss / unexpected state)
  → Automation Agent: exhaust retry budget (up to 3 attempts with backoff)
  → Orchestrator: activate Investigation Agent
  Investigation Agent:
    ① pull artifact bundle from Evidence Engine (pre/post/HAR/console/diff)
    ① Libretto: run executeRecoveryAgent for immediate in-session fix attempt
    ② if recovery fails: pull artifact bundle from Evidence Engine + prior observation from Run Store
    ③ Investigation Agent (pi-agent-core Agent → pi-ai): InvestigationRequest → InvestigationReport
    ④ classify failure category + generate structured fix proposal
  → if report.requiresHumanReview:
      Orchestrator: pause run → push to review queue with report + artifacts
      Human: reviews side-by-side, approves or overrides fix
  → else:
      Orchestrator: auto-apply fix if suggestedFix.confidence > 0.85
  → fix applied:
      Run Store: create workflow v+1 with patched steps
      Knowledge Agent: ingest failure report + fix as embeddings
      Knowledge Agent: update selector_history for affected selector
      Knowledge Agent: recompute dom_fingerprint baseline if page structure changed
      Orchestrator: resume run from failed step using new version
      run continues on happy path →
```

### Proactive Drift Detection (between runs)

```
Automation Agent: step pre-check → fingerprint current page
  → fingerprint Jaccard vs. stored baseline < DRIFT_THRESHOLD
  → Orchestrator: activate Investigation Agent before executing action
  Investigation Agent:
    ① fetch last-known dom_fingerprint baseline from Knowledge Agent
    ② compute DOMDiff: current snapshot vs. baseline
    ③ generate drift report: "nav element relocated, 3 link labels changed"
  → if diff affects this step's selector:
      → queue step for human review with drift context
  → if diff does not affect this step's selector:
      → update baseline, log drift as informational, continue
```

---

## Self-Improvement Loop (Detailed)

Each resolved failure moves through this pipeline:

```
failure_report.resolution = 'accepted' | 'auto_healed'
  ↓
Knowledge Agent: ingest_failure(report)
  1. Embed: root_cause + selector context + page URL → insert into failure_report vectors
  2. Update selector_history:
       old_selector: decrement success_rate
       new_selector: insert with initial success_rate = 1.0
  3. Site profile: if category = 'unexpected_modal' → append to known_quirks
  4. DOM baseline: if fix required re-locating element → update dom_baseline for this page
  5. Recompute confidence factors for similar steps across all active workflows
       → any step with selector matching old_selector: flag for re-scoring
  ↓
Next run of any workflow targeting this site:
  - Automation Agent queries Knowledge Agent: "similar element on this page?"
  - Returns new_selector with high confidence (drawn from recent fix)
  - Step score improves → fewer human reviews required over time
```

The compounding effect: a site that initially requires 4 human reviews per workflow run converges toward 0 as the Knowledge Agent accumulates selector mappings, DOM baselines, and known quirk patterns.

---

## Browser Pool Management

Playwright sessions are expensive to spin up. The platform maintains a warm pool.

```typescript
interface PoolConfig {
  minIdle: number;        // default 2 per browser type
  maxPerBrowserType: number;  // default 10
  sessionTTL: number;    // ms; default 300_000 (5 min) — recycle after idle
  browserTypes: ('chromium' | 'firefox' | 'webkit')[];
}

// Sessions are checked out per run, checked back in on completion/failure
// If a session crashes (browser process killed), pool auto-replenishes
// Sessions are isolated: no cookies/storage bleed between runs
```

---

## Operational Concerns

### Rate Limiting
- Per-domain: configurable request rate cap (default 2 req/s) enforced inside Browser Action API
- Per-run: max concurrent browser sessions per tenant (default 5)
- AI API: token budget tracked per run; Investigation Agent calls are the largest consumers

### Secrets / Credentials
- Credentials for `auth_wall` handling are stored in an external vault (Hashicorp Vault or AWS Secrets Manager)
- The Browser Action API retrieves them at runtime; they are never embedded in workflow steps or logs
- `sensitiveFields` list in planner constraints causes `fill` values to be masked in all stored observations

### Audit Log
Every action, agent decision, human override, and workflow version change is appended to an append-only `audit_events` table:

```sql
CREATE TABLE audit_events (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  run_id      UUID,
  actor       TEXT,   -- 'automation_agent' | 'investigation_agent' | 'orchestrator' | 'human:{userId}'
  event_type  TEXT,
  payload     JSONB,
  occurred_at TIMESTAMPTZ DEFAULT now()
);
```

### Observability
- Structured JSON logs (pino) shipped to log aggregator; each log line carries `run_id`, `step_id`, `agent`
- OpenTelemetry traces: one root span per run, child spans per step and per agent call
- Metrics (Prometheus):
  - `run_pass_rate` by site
  - `steps_requiring_review_rate` by site
  - `mean_confidence_score` over time (should rise as Knowledge Agent learns)
  - `investigation_agent_calls_per_run` (should fall as self-improvement takes hold)
  - `browser_session_pool_utilization`

---

## Package Structure (Monorepo)

```
falcona/
├── apps/
│   ├── web/                    # React + TanStack Router UI
│   └── api/                    # Hono API server
├── packages/
│   ├── agents/
│   │   ├── orchestrator/       # pi-agent-core Agent: planner, state machine
│   │   ├── automation/         # pi-agent-core Agent: step executor, branch handler
│   │   ├── investigation/      # pi-agent-core Agent: forensics, root cause, fix proposal
│   │   └── knowledge/          # pi-agent-core Agent: vector search, self-improvement
│   ├── browser/
│   │   ├── action-api/         # BrowserActionAPI — wraps Libretto SDK
│   │   ├── pool/               # BrowserSession pool manager (wraps Libretto launchBrowser)
│   │   └── evidence-engine/    # Artifact capture + S3 storage (uses Libretto instrumentPage)
│   ├── run-store/              # Drizzle schema + repository: workflows, runs, observations
│   ├── queue/                  # BullMQ job definitions + workers
│   └── types/                  # Shared TypeScript types (contracts above)
├── infra/
│   ├── docker-compose.yml      # Local: Postgres + Redis + S3 (MinIO)
│   └── migrations/             # Drizzle migration files
└── turbo.json
```

---

## Non-Goals

- Not a test framework. Tests are a downstream consumer of workflows — export a workflow as a Playwright test file if needed, but test authoring is not a platform concern.
- Not a low-code recorder. Recorded actions are a seed; the AI layer is responsible for making them durable and self-healing.
- Not a web scraper. The focus is workflow automation on authenticated, interactive applications — not bulk data extraction from public sites.
- The AI layer does not replace human judgment for ambiguous or high-risk actions. It surfaces them with evidence, a structured root cause, and a recommended course of action — then waits.

---

## Open Questions (Decision Required Before Build)

1. **Multi-tenancy model**: single Postgres database with `tenant_id` columns, or schema-per-tenant? Schema-per-tenant simplifies isolation and backup; single-schema is simpler to operate at low scale.
2. **Workflow step DSL**: JSON step objects (current) vs. a domain-specific language (more expressive, harder to parse). JSON is AI-friendly; a DSL may be more readable for human editors.
3. **Replay isolation**: should a replay run against the live site, a recorded HAR replay (no network), or a Playwright HAR route intercept? Live is most realistic; HAR replay is fully deterministic and fast for debugging.
4. **Confidence threshold values**: the `0.85 / 0.60` thresholds above are starting estimates. These should be tunable per workflow and per site — some sites warrant stricter review gating than others.
5. **Libretto recovery agent boundary**: Libretto's `executeRecoveryAgent` uses its own model config (`COMPUTER_USE_RECOVERY_MODELS`). Decide whether to let Libretto drive the first recovery attempt autonomously, or always route through the Investigation Agent so all recovery decisions are tracked in the Run Store and observable in the UI.
