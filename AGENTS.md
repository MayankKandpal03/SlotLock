# AGENTS.md — SlotLock agent instructions

## Mission and scope

Build a production-style FastAPI booking backend from `SPEC.md`. Prioritize correctness, explainability and reproducibility over feature count. Follow `TASKS.md`; keep `PROGRESS.md` current. The specification is authoritative for product behavior; its proposed clarifications are labeled separately. Preserve existing user work. Do not silently change the API, stack, budget or MVP boundaries.

## Start every session

1. Read this file, `SPEC.md`, `TASKS.md`, and `PROGRESS.md` in full.
2. Inspect `git status --short`, `git log -5 --oneline`, the working tree, current branch, migrations and existing tests. Treat notes as claims to verify, not proof.
3. If a checkpoint is stale, reconcile it against the code and actual test results. Resume the first unfinished, unblocked task in dependency order. Never reset the repository or restart from scratch.
4. Record a short plan and any decisions from the SPEC clarification register that affect this task. Work in small, coherent slices; continue to the next independent task if blocked.

## Architecture

```text
app/
  core/          settings, logging, security, errors
  domain/        pure slots, pricing, cancellation, decide()
  services/      use cases, authorization, transaction boundaries
  repositories/  typed Protocols + SQLAlchemy implementations
  api/           FastAPI routers, Pydantic schemas, dependencies
  infra/         Redis, hold/limiter/idempotency adapters, rule loading
migrations/     Alembic migrations
tests/         unit, integration and API tests
k6/             ES6+ load scripts
```

- Dependency direction: API -> services -> domain and repository protocols. Inject concrete adapters; keep domain free of network, database, environment and wall-clock I/O.
- Use type hints, clear names, cohesive modules and explicit transaction ownership. Prefer simple functions and data structures to unnecessary frameworks. Use consistent synchronous or asynchronous I/O boundaries; do not block async request handlers with synchronous I/O.
- Use `uv`, a committed lockfile, ruff, mypy, pytest and Testcontainers. Keep settings in pydantic-settings and secrets in the environment. Keep sample environment values non-secret.
- Routers parse and serialize; services enforce business rules and scope. Repositories execute data access; domain functions calculate deterministic decisions. Never implement critical checks only in a router.

## Non-negotiable correctness

- PostgreSQL `btree_gist` and a partial exclusion constraint on `(court_id, during)` for confirmed rows are mandatory. Never use SELECT-then-INSERT, application locks, Redis or process-local locks as the final overlap guarantee.
- Use nonempty half-open ranges, aligned slots, UTC instants and venue-local time for opening/pricing rules. Money is integer paise. Avoid binary floating point money arithmetic.
- Use transaction-scoped advisory locks for idempotency and a unique `(user_id,key)` index. Replay before checking a consumed hold. Hash method/path/body; same-key changed request is 422; concurrent execution is 409. Atomic booking and response persistence; rollback cleanly or use savepoints for terminal conflicts.
- Redis holds are advisory, five-minute TTL, atomic acquire and owner-safe Lua release. Do not delete a successor's hold. Validate hold authorization and expiry even in degraded mode. Fail open only on a verified Redis infrastructure error as described in SPEC.
- Scope OWNER/STAFF to venue and CUSTOMER to own booking; STAFF belongs to one venue. Prevent IDOR and privilege escalation. Check authorization on every relevant object, including degraded paths and retries.
- Argon2, short-lived JWTs, hashed refresh tokens, atomic rotation/reuse detection and family revocation. Never log tokens, passwords, secrets or customer phone/name. No secrets in Git, sample files, logs or test output.
- Versioned rules are immutable; publish validated YAML safely (no arbitrary code execution), preserve rule version and quote, log non-sensitive reasons. No real payment or refund integration in MVP.
- Return `application/problem+json`, request IDs, stable keyset pagination, correct status codes and Retry-After for rate limiting. Log JSON, expose internal Prometheus metrics with bounded-cardinality labels. `/readyz`: PG down 503; Redis-only outage 200 degraded.

## Development workflow

1. Implement only the next small task, adding/updating tests alongside it. Use migrations for all schema changes; do not rely on ORM auto-create in production.
2. Run focused tests, then relevant integration checks. Standard commands (adjust only to actual project configuration and document changes):
   ```sh
   uv sync --locked
   uv run ruff check .
   uv run ruff format --check .
   uv run mypy app
   uv run pytest -q
   docker compose config
   ```
3. When available, run real PostgreSQL/Redis tests, migrate a fresh test DB, boot Compose, and exercise health/readiness. Do not substitute mocks for correctness tests or weaken assertions to pass CI. Record exact commands, results and environment limits.
4. Update `TASKS.md` only for verified acceptance criteria. Update `PROGRESS.md` with completed work, pending work, decisions, blockers, test output summary and the next exact step.
5. Create a small descriptive Git commit when permitted; record its hash if available (or note uncommitted changes / platform-managed patch). Checkpoint files must travel with code. Do not claim cloud state is durable unless changes are actually committed/pushed or preserved by the platform.
6. Continue to the next unblocked task while execution allowance permits. Do not wait for confirmation for routine in-scope coding. If interrupted, the next agent must be able to recover using the files and Git. Never bypass usage limits or claim this file schedules an automatic retry.

## Required verification

- N concurrent confirmations: one 201, remaining 409, zero confirmed overlaps, including with Redis down. Use independent DB connections and synchronize contenders; use distinct idempotency keys.
- Same-key replay after commit and lost response: one booking; same key changed request 422; in-flight duplicate 409. Cancellation retries cannot duplicate refunds.
- Pricing/refund boundary cases, adjacent intervals, misaligned slots, immutable old rules and quote behavior.
- Full authorization matrix across roles, own/foreign venue and own/foreign booking; anonymous access and invalid credentials.
- Redis expiry, stale release, rate limit, recovery and degraded readiness. Refresh rotation, reuse, expiry and concurrent refresh behavior.
- No fabricated test results or load measurements. Record unavailable infrastructure as a blocker and leave relevant tasks unchecked.

## Git, deployment and safety boundaries

- Work on the task branch; never force-push, rewrite history, discard user changes, merge to main, enable paid resources or deploy to production without approval. Do not alter unrelated repositories.
- Implement CI/CD configuration, GHCR publish and SSH deployment workflow as scoped. Gate actual deployment on configured secrets and explicit owner approval; never invent credentials or print secrets.
- Keep stretch work out of MVP. Do not add AI, microservices, Kubernetes, Terraform or large frameworks. Maintain the source's free-tier / $10 monthly target and time budget; ask for approval before expanding scope or cost.
- Do not contact venue owners or claim real-world demos on the user's behalf. Prepare demo instructions; mark owner validation pending until supplied.

## Definition of done

All in-scope task gates are verified, README explains setup and correctness tradeoffs with Mermaid diagram, RESULTS.md contains genuine measured data with test environment details, CI is green if observable, and remaining operational/manual prerequisites are explicitly listed. If external deployment or owner validation remains blocked, report MVP implementation separately from full project completion.
