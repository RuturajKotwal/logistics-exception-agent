# Agent Instructions

This file gives AI coding agents repository-wide context and guardrails. Read it before changing project files. Treat `architecture.md` as the system design and the project TODO checklist under `docs/todos.md` as the scope/milestone plan. User requests in the active conversation take precedence over project notes when they conflict.

## Project purpose

Build a local, Slack-native logistics exception demo. A driver reports a delay; the system parses the report, finds a feasible dock slot deterministically, assesses impact, applies deterministic policy, and either auto-executes a low-risk change or requests dispatcher approval. Every exception should have an audit trail. The LLM is a language component, not an authority over scheduling or policy.

The intended v1 is fictional-data/local-demo software. It is not a production logistics system. Do not expose the unauthenticated web console or API on a network or use them with real operational data.

## Source of truth and scope

- Read `architecture.md` before changing behavior. It defines interfaces, state transitions, data, policy, guardrails, and non-goals.
- Read the TODO checklist before starting a milestone. Complete its check before moving to the next milestone where practical.
- Do not silently change a documented policy, state transition, API contract, or safety invariant. Update the relevant design/checklist documentation when an approved change requires it.
- Avoid features beyond the requested milestone and the v1 scope. Do not interpret scope guidance as permission to ignore an explicit user request.
- Preserve unrelated user changes and untracked files. Inspect `git status` before broad edits; never reset, clean, or overwrite work to make the checkout look tidy.

## Stack and repository layout

The planned stack is TypeScript throughout: Node.js 20+, Fastify API, Slack Bolt with Socket Mode, PostgreSQL with Drizzle or Prisma, shared Zod schemas, Ollama through its OpenAI-compatible API, React/Vite console, Pino logging, Vitest, and GitHub Actions.

The architecture describes this intended layout:

```text
/shared       Shared Zod schemas and TypeScript types
/api          Fastify API, Slack adapter, LLM client, domain, pipeline, executor, audit, DB
/web          React/Vite console
/eval         Scenario corpus, runner, and generated evaluation reports
architecture.md
AGENTS.md
```

The repository may still be at the planning stage. Before assuming a package manager, scripts, or source layout, inspect the actual files and lockfiles. Use the package manager already selected by the repository. If none is selected yet, use the project setup milestone's choice and commit only that manager's lockfile; do not create competing npm/pnpm/yarn lockfiles. Do not invent commands such as `npm run build` unless a matching script exists. Document any newly added developer commands in the README or the relevant checklist.

## Development and environment

- Use Node.js 20 or newer as specified by the architecture. Check the installed version and package scripts before running commands.
- PostgreSQL is intended to run locally through Docker Compose. Do not assume Docker is installed or running; report a missing local prerequisite rather than changing deployment assumptions.
- Keep real credentials in the ignored backend `.env`. `.env.example` contains names/placeholders only. Never print, copy, commit, or place secrets in frontend code, logs, screenshots, eval reports, or generated docs.
- The Ollama/JupyterHub endpoint and university GPU are external dependencies. The `llama3.1:8b` path is the required default; the 32B comparison is conditional on GPU access and time. Never fabricate results or make the 8B path depend on the 32B model.
- Keep tokens server-side. The browser must never receive the LLM token, Slack bot/app token, signing secret, or database credentials.
- Keep the demo local. Socket Mode avoids a public Slack event URL, but it does not secure the HTTP API or console.

## Architecture and coding rules

### Trust boundaries and LLM

- LLM calls are limited to `parseReport` and `draftNotification`. Do not add tools, autonomous loops, scheduling authority, policy decisions, or database mutation to the model.
- Validate external and model outputs at boundaries with Zod. A valid schema or known truck ID does not prove the extraction is factually correct.
- Unknown, conflicting, or materially ambiguous truck/delay details must lead to clarification. Bound delay values and verify referenced entities against the database.
- Use structured output for parsing, one bounded retry for invalid output, then clarification or a deterministic text-template fallback as specified in the architecture.
- Keep prompts and outputs free of secrets. Record model, latency, token counts, validity, and exception correlation as defined by the data model.

### Deterministic scheduling and policy

- Slot finding, impact assessment, and approval policy are deterministic code. Never delegate these choices to an LLM.
- Appointment intervals are half-open `[start, end)`. Apply configured buffers symmetrically and use the same validation for search, dispatcher edits, and the final execution check.
- Interpret operating shifts using the warehouse's explicit IANA timezone; persist/compare timestamps consistently as UTC instants. Never fall back to the machine's local timezone. Missing timezone data fails closed.
- In v1, do not silently roll a slot into another local operating day. No valid same-day slot means no proposal and dispatcher handling.
- Policy thresholds and risk rules belong in the policy module/config, not Slack handlers, the web UI, prompts, or ad hoc route code. Always provide visible reasons for approval decisions.

### State, persistence, and side effects

- Keep pipeline transitions explicit and auditable. Persist audit/trace entries for state changes and errors.
- Executor operations must be idempotent by exception and re-check slot availability inside the database transaction before changing an appointment.
- A database commit and Slack delivery are not atomic. Persist notification delivery state; after a post-commit failure, preserve the committed appointment, show the failure, and retry notification idempotently without applying the appointment mutation again.
- Handle duplicate Slack events and repeated approval clicks safely. Validate Slack actions and API requests at their boundaries; do not rely on the UI to enforce business rules.
- Use migrations for schema changes and keep migration behavior explicit and reviewable. Do not edit or delete migration history to conceal a schema change.

### Code style

- Use TypeScript and the existing formatter/linter conventions once established. Prefer explicit types at module boundaries and shared Zod schemas for runtime-validated data.
- Use clear, domain-specific names (`truckId`, `exceptionId`, `newStart`) consistently across API, DB, and UI.
- Keep functions and modules focused. Prefer straightforward code over speculative abstractions, generic frameworks, or extra dependencies.
- Validate at trust boundaries (Slack/API input, model output, database writes); avoid redundant validation in trusted internal calls unless it protects an invariant.
- Comments should explain non-obvious reasoning or safety constraints, not restate the code.
- Keep frontend code presentational where possible. Business rules belong in shared/domain/API services, not duplicated in React or Slack callbacks.
- Do not add a runtime dependency for a convenience wrapper when a small, clear implementation using existing dependencies is sufficient. Explain material dependency additions in the change summary.

## Testing, linting, and CI

Use the scripts actually declared in the root/workspace manifests. The planned CI checks are `typecheck`, `lint`, `test`, plus a small mocked-LLM eval subset. Do not claim a check passed unless it was run and its result observed.

When implementation work is requested, add focused tests alongside changed behavior when the task/checklist calls for them. In particular:

- Slot finder tests cover no double booking, shift boundaries, symmetric buffers, half-open endpoint behavior, warehouse-local/UTC date boundaries, missing timezone failure, and no-slot behavior.
- Policy tests cover perishables, at-risk/priority shipments, delay thresholds, and human approval reasons.
- LLM tests mock the model and cover valid parsing, retry, clarification on ambiguity/unknown IDs, draft fallback, and prompt-injection-style input.
- Pipeline/executor tests cover state transitions, idempotency, duplicate events/actions, concurrent slot re-checks, audit records, and notification failure/retry without duplicate appointment mutation.
- API/UI checks cover loading/error states and ensure approval actions use the same service and validation path.
- Evals report extraction and clarification metrics separately from deterministic slot invariants. A zero-invalid-proposal result applies only to the fixed evaluated scenarios; do not describe it as a general guarantee.

Before wrapping implementation work, run the narrowest relevant checks, then the repository's declared lint/typecheck/test commands when time and environment allow. If a command cannot run because a service, credential, GPU, or external cluster is unavailable, say exactly what was not run and why. Never weaken or delete a test just to make CI green; fix the implementation or explain a justified expectation change.

## Explicit never-do rules

- Never read, print, commit, copy, or expose `.env` secrets. Do not put credentials in browser bundles, logs, fixtures, screenshots, or reports.
- Never let the LLM choose a dock slot, assess policy, approve a change, or execute a database/Slack action.
- Never execute an appointment without deterministic slot validation and the required policy/dispatcher decision.
- Never claim the unauthenticated console is safe for public/shared use or real operations. Do not add public deployment as an incidental change.
- Never silently use host-local time when warehouse timezone is missing, silently book into another day, or bypass the final availability re-check.
- Never treat schema-valid model output, passing a small eval set, or zero invalid eval proposals as proof of general correctness or safety.
- Never create or commit real Slack messages, external notifications, public uploads, or other irreversible external side effects unless the user explicitly requested that action.
- Never add the architecture's listed non-goals (Teams, voice, multi-tenancy, RBAC/auth, real TMS/WMS integrations, RAG, autonomous multi-step agent loops, public deployment, rich dashboards, mobile) without an explicit scope change.

## Reporting changes

In the final summary, state what changed, why, which checks were run and their outcome, and any environment limitation or material risk. Link repository files using their absolute paths when available.
