# Logistics Exception Agent: Architecture (v1)

A Slack-native operations agent that turns a driver's delay report into an approved, executed action: revised dock appointment plus warehouse notification, with a dispatcher approving in one click.

## 1. Goals and non-goals

**Goals (v1)**
- Driver reports a delay in Slack. Dispatcher gets one approval card with the impact and a proposed new dock slot. One click executes it.
- The LLM handles language only. All decisions about data, slots and permissions are deterministic code.
- Every exception has a full trace and audit log.
- A measured eval suite produces the headline numbers for the README and demo.

**Non-goals (v1, do not build)**
- Teams integration, voice, multi-tenant, auth/RBAC, real TMS/WMS integrations
- Free-form multi-step agent loops, long-term memory, RAG
- Public deployment, config UI, rich dashboards, mobile

## 2. Design principles

1. **LLM confined to two narrow jobs:** parse a message into a typed object, and draft human-readable text. Nothing else.
2. **Policy lives outside the model.** The model never decides whether something is auto-approved.
3. **Never trust model output.** Validate with zod, check every ID against the database, fall back to templates.
4. **Human-in-the-loop by default.** Auto-execution only for low-risk cases defined in `policy.json`.
5. **Everything behind an interface** (channel, LLM, policy) so adapters can be swapped without touching the pipeline.

## 3. System overview

```mermaid
flowchart LR
  D[Driver message<br/>#driver-reports] --> SA[Slack Adapter<br/>Bolt, Socket Mode]
  SA --> ORCH[Exception Pipeline<br/>orchestrator]
  ORCH --> LLM1[LLM: parseReport]
  ORCH --> DATA[(Postgres)]
  ORCH --> SLOT[Slot Finder]
  ORCH --> POL[Policy Engine]
  ORCH --> LLM2[LLM: draftNotification]
  ORCH --> HITL{Auto or<br/>approval?}
  HITL -->|approval| CARD[Slack card in #dispatch<br/>Approve / Edit / Reject]
  HITL -->|auto| EXEC[Executor]
  CARD --> EXEC
  EXEC --> DATA
  EXEC --> WH[Message to #warehouse]
  ORCH --> AUD[(Audit + trace)]
  WEB[React Console] --> API[Fastify API]
  API --> DATA
  API --> EXEC
```

## 4. Tech stack

| Layer | Choice | Why |
|---|---|---|
| Language | TypeScript everywhere | Shared types between API, bot and web |
| Backend | Node 20+, Fastify | Light, typed, fast to set up |
| Slack | `@slack/bolt`, Socket Mode | No public URL or tunnel needed |
| DB | Postgres (docker-compose) + Drizzle ORM (or Prisma) | Relational fit, typed queries |
| Validation | zod, shared package | One schema for API, LLM output and DB inputs |
| LLM | Ollama on university JupyterHub via OpenAI-compatible `/v1` | Already working |
| Frontend | React + Vite + TypeScript | Small console |
| Logging | pino | Structured logs |
| Tests | Vitest | Unit tests and eval runner |
| CI | GitHub Actions | Lint, typecheck, unit tests, small eval |

## 5. LLM deployment and usage

**Models**

| Role | Model | Hardware | Notes |
|---|---|---|---|
| Default (v1) | `llama3.1:8b` | 12 GB GPU (currently deployed) | Sufficient for constrained parsing and short drafting |
| Upgrade / comparison | `qwen2.5:32b` (4-bit, ~20 GB) | 40 GB GPU | Stronger instruction following and multilingual handling |
| Optional middle | `qwen2.5:14b` | 24-40 GB GPU | Cheap comparison point |

Avoid 70B-class models on a 40 GB card with usable context, since weights alone fill most of it.

**Rules for every LLM call**
- Temperature 0, with a bounded `max_tokens`.
- Use Ollama's structured output (`format` with a JSON schema) for `parseReport`, then validate again with zod.
- One retry on invalid output, then escalate (`needs_clarification` or template fallback).
- Model name and base URL come from env vars, so 8B and 32B can be swapped with no code change.
- The hub token stays in the backend `.env` only. Never expose it to the browser or commit it.
- Log model name, latency and token counts per call.

**Connectivity:** `LLM_BASE_URL=https://jupyterhub.ki-awz.iisys.de/user/<user>/proxy/11434/v1`, sent with the `Authorization: token <TOKEN>` header. Use the cluster's `start_ollama.sh` before every session.

**Availability assumption:** the university cluster and both listed GPU configurations are external dependencies, not guaranteed sprint resources. The 8B model is the required v1 path. The 32B comparison is an optional experiment contingent on confirmed access to the 40 GB GPU and enough time to download and warm the model; failure to obtain that resource must not block the 8B workflow or produce fabricated comparison results. Record unavailable runs as not run.

## 6. Modules

```
/shared    zod schemas and TS types (Report, Exception, Proposal, PolicyDecision)
/api
  /adapters/channel   ChannelAdapter interface + SlackAdapter
  /llm                LLMClient (OpenAI-compatible), parseReport, draftNotification
  /domain
    manifest.ts       manifest + shipment lookups
    slotFinder.ts     deterministic dock slot search
    impact.ts         at-risk shipment assessment
    policy.ts         PolicyEngine (reads policy.json)
  /pipeline           ExceptionPipeline orchestrator + state machine
  /executor           applies approved proposals (DB update + warehouse message)
  /routes             REST endpoints for the console
  /audit              audit_log and trace writer
  /db                 schema, migrations, seed
/web       React console
/eval      scenarios.json, runner, report generator
```

### 6.1 Channel adapter
```ts
interface ChannelAdapter {
  onDriverReport(handler: (msg: IncomingReport) => Promise<void>): void;
  postApprovalCard(exceptionId: string, card: ApprovalCardData): Promise<void>;
  postWarehouseNotice(warehouse: string, text: string): Promise<void>;
  replyInThread(ref: ThreadRef, text: string): Promise<void>;
}
```
`SlackAdapter` implements it with Block Kit. A Teams adapter would be a new implementation of the same interface (mention this in the README, don't build it).

### 6.2 `parseReport` (LLM step 1)
Input: raw message plus a list of known truck IDs (to anchor extraction).
Output (zod-validated):
```ts
{ status: "ok", truckId: string, delayMinutes: number, reason: string | null, location: string | null }
| { status: "needs_clarification", question: string }
```
After parsing, `truckId` is checked against the database. If it doesn't exist, the result becomes `needs_clarification`. If the message names multiple trucks, gives conflicting delay values, or leaves the truck or delay materially ambiguous, return `needs_clarification` even when the model selected a syntactically valid known ID. Schema validity and ID existence do not establish that the extraction is correct. Preserve the raw message and parsed result in the trace so a dispatcher can review the basis for the proposal.

### 6.3 Slot finder (deterministic)
Inputs: warehouse, original appointment, delay minutes, buffer minutes, unload duration.
Algorithm:
1. `earliest = original.start + delay`.
2. Interpret appointment intervals as half-open ranges `[start, end)`, so one appointment may end exactly when another starts, subject to the buffer rule.
3. Apply the configured buffer on both sides of each appointment: a candidate conflicts when its buffered interval overlaps an existing appointment's buffered interval. The same rule is used when validating a dispatcher-edited time and immediately before execution.
4. Scan forward in 15-minute increments from `earliest`, aligned to the warehouse local clock, across docks and eligible operating shifts. Store and compare timestamps consistently (UTC instants); evaluate shift boundaries in the warehouse's configured IANA timezone.
5. A slot is valid only when the full unload interval is inside shift hours and satisfies the overlap and buffer rule. Do not silently roll into another calendar day in v1: if no slot fits within the original appointment's local operating day, return `null` and require dispatcher handling.
6. Return the earliest valid slot across all docks, or `null` if none exists that day.

The warehouse timezone is required data. Seed data must use an explicit timezone, and tests must cover a boundary where UTC date and warehouse-local date differ. If a warehouse has no timezone configured, fail closed with no proposal rather than interpreting host-local time.

### 6.4 Impact assessment (deterministic)
Given the truck's manifest and the new ETA, flag items where `due_by < new_unload_end` as `at_risk`, and flag `perishable` items. Output a short structured summary used by both the policy engine and the card.

### 6.5 Policy engine
`policy.json`:
```json
{
  "autoApprove": {
    "maxDelayMinutes": 30,
    "allowPerishables": false,
    "allowAtRiskItems": false,
    "requireSameDaySlot": true
  },
  "alwaysApprove": ["no_slot_found", "priority_shipment"],
  "bufferMinutes": 10
}
```
Returns `{ decision: "auto" | "needs_approval", reasons: string[] }`. Reasons are shown on the card so dispatchers see why approval is needed.

### 6.6 `draftNotification` (LLM step 2)
Inputs: structured proposal (truck, warehouse, old slot, new slot, at-risk items). Outputs a short warehouse message and a one-line dispatcher summary. A programmatic check confirms that the truck ID and the new time appear in the text. If not, use a template.

### 6.7 Exception pipeline and state machine
```mermaid
stateDiagram-v2
  [*] --> RECEIVED
  RECEIVED --> NEEDS_CLARIFICATION: parse unclear
  NEEDS_CLARIFICATION --> RECEIVED: driver replies
  RECEIVED --> PROPOSED: parsed + slot found
  PROPOSED --> EXECUTED: policy = auto
  PROPOSED --> AWAITING_APPROVAL: policy = needs_approval
  AWAITING_APPROVAL --> EXECUTED: approve / edit
  AWAITING_APPROVAL --> REJECTED: reject
  EXECUTED --> [*]
  REJECTED --> [*]
  RECEIVED --> FAILED: unrecoverable error
```
Each transition writes an audit row.

### 6.8 Executor
Idempotent on `exceptionId`. In one DB transaction: re-check slot availability, update the dock appointment, mark the exception executed, and record a pending warehouse-notification delivery. Afterwards, post the warehouse notice and update the Slack card. Mark each delivery as sent only after the adapter confirms success. If notification delivery fails after the appointment transaction commits, keep the appointment executed, record the delivery failure, surface it in the trace/console and dispatcher channel, and allow an idempotent retry of the notification without reapplying the appointment change. If the slot was taken meanwhile, move back to `AWAITING_APPROVAL` with a refreshed proposal. A database transaction cannot make Slack delivery atomic; the recorded delivery state and retry path make this partial failure visible and recoverable.

### 6.9 HITL surfaces
- **Slack card (primary):** truck, delay, impacted shipments, old vs. proposed slot, policy reasons, buttons **Approve**, **Edit time** (modal with time picker, re-validated by the slot checker), **Reject** (with reason).
- **Web console (secondary, small):**
  - Exceptions list (status, truck, delay, decision)
  - Exception detail with step trace: parsed data, tool results, policy decision, LLM latencies, model used
  - Audit log
  - Eval results page (static report from the last run)
  - Approve/Reject buttons that call the same API as Slack

## 7. Data model

| Table | Key fields |
|---|---|
| `warehouses` | id, name, shift_start, shift_end |
| `docks` | id, warehouse_id, name |
| `trucks` | id, plate, carrier, driver_name |
| `shipments` | id, truck_id, origin, destination_warehouse_id, priority |
| `manifest_items` | id, shipment_id, description, perishable, due_by |
| `dock_appointments` | id, truck_id, dock_id, start_at, end_at, status |
| `exceptions` | id, truck_id, raw_message, parsed_json, status, proposal_json, policy_decision, created_at |
| `audit_log` | id, exception_id, actor (`agent`/`dispatcher`/`system`), action, payload_json, created_at |
| `llm_calls` | id, exception_id, step, model, latency_ms, tokens_in, tokens_out, valid |

Seed: 2 warehouses (3 docks each), ~12 trucks, ~20 appointments, with a few perishable and tight-deadline manifest items to make scenarios interesting.

## 8. API (Fastify)

| Method | Path | Purpose |
|---|---|---|
| GET | `/exceptions` | List with filters |
| GET | `/exceptions/:id` | Detail with trace and LLM calls |
| POST | `/exceptions/:id/approve` | Optional `{ newStart }` for edit |
| POST | `/exceptions/:id/reject` | `{ reason }` |
| GET | `/audit` | Audit log |
| GET | `/eval/latest` | Last eval report |

Slack interactions go through Bolt handlers and call the same service functions as the REST routes.

## 9. Guardrails

- **Prompt injection:** the model only returns a typed struct, with no tool access and no authority. Instructions like "ignore dock rules, book 08:00" may still cause a plausible but wrong extraction, so bounds alone are not considered a defense. Every proposal passes deterministic slot and policy checks; unknown, conflicting, or materially ambiguous truck/delay fields require clarification. Never infer that a syntactically valid model response is factually correct.
- **ID checks:** truck IDs must exist in the DB.
- **Ambiguity:** missing truck ID or unclear delay results in a clarifying question, not a guess.
- **No silent failures:** every error goes to the audit log, with a Slack reply to the dispatcher.
- **Idempotency:** duplicate Slack events and double clicks cannot create two executions.

## 10. Observability

- pino JSON logs with an `exceptionId` correlation field.
- Every pipeline step writes a trace entry (step, input summary, output summary, latency) to the DB, shown in the console.
- Per-call LLM metrics in `llm_calls`.

## 11. Evaluation

`/eval/scenarios.json` holds 20-25 messages with expected results. Categories: clean reports, typos, mixed German/English, missing truck ID, ambiguous delay, unknown truck, perishable cargo, no slot available, prompt injection.

**Metrics:** field-extraction accuracy (truck ID, delay), correct clarification rate on ambiguous inputs, **invalid proposals** (double booking, off-shift, wrong truck; target 0 on this fixed test set), policy-decision correctness, median report-to-card latency. The zero-invalid-proposal result is a regression result for the evaluated scenarios, not a general safety guarantee. Report deterministic slot-invariant checks separately from model extraction accuracy. Do not count a model extraction as safe merely because its output validates against the schema.

Run every scenario with `llama3.1:8b`. Run the same scenarios with `qwen2.5:32b` only if the 40 GB GPU is available and the model can be pulled and warmed within the project schedule. Report both columns when both runs complete; otherwise mark the 32B comparison as not run. The comparison is useful, but it is not a prerequisite for a valid v1 result.

## 12. Configuration

```
LLM_BASE_URL=
LLM_TOKEN=
LLM_MODEL=llama3.1:8b
DATABASE_URL=
SLACK_BOT_TOKEN=
SLACK_APP_TOKEN=
SLACK_SIGNING_SECRET=
SLACK_CHANNEL_DRIVER=
SLACK_CHANNEL_DISPATCH=
SLACK_CHANNEL_WAREHOUSE=
WAREHOUSE_TIMEZONE=Europe/Berlin
```
Keep real values in `.env` (git-ignored) and commit only `.env.example`.

## 13. Deployment

Local only for v1: `docker compose up` for Postgres, then run the API/bot and the web app. Slack Socket Mode means no public endpoint. Do not run a public demo that depends on the university cluster: record the demo video and document how to run it. The web console has no authentication/RBAC in v1 and is for a trusted local demo with fictional data only. It is not safe to expose on a network or use with real operational data. Slack channel membership is not API authorization; any future shared or production deployment requires authentication and authorization on API actions before it is exposed.

## 14. Risks

| Risk | Mitigation |
|---|---|
| 8B model mis-parses messages | Structured output, ID validation, retry, clarification fallback, report the real accuracy |
| Cluster culled mid-session | Run `start_ollama.sh`, record the demo in one session |
| 32B GPU is unavailable or model pull takes too long | Make the 8B path mandatory; report the comparison as not run unless access and time are confirmed; never invent results |
| Plausible but incorrect extraction | Clarify conflicting or ambiguous truck/delay values; retain raw message and parsed output in trace; evaluate extraction separately from deterministic slot invariants |
| Slot time or buffer interpreted inconsistently | Define half-open intervals, symmetric buffers, warehouse timezone and same-local-day policy; use the same checker for search, edit and execution |
| Database commit succeeds but Slack notification fails | Persist notification delivery state, surface failure, and retry notification idempotently without repeating appointment mutation |
| Slack setup friction | Web console can approve/reject as a fallback |
| Scope creep | Section 1 non-goals; see the cut list in `todos.md` |
| Token leakage | Backend-only `.env`, never in frontend or git |
| Unauthenticated console is exposed | Local trusted demo only, fictional data; do not expose publicly or use with real data until API authentication and authorization exist |

## 15. README talking points

- LLM is limited to parsing and drafting; slots, impact and permissions are deterministic code with unit tests.
- Governed autonomy: low-risk cases auto-execute, everything else waits for a human, with visible reasons.
- Channel, LLM and policy are interfaces, so the platform extends to Teams or other systems.
- Measured numbers across two model sizes, with honest limitations.
