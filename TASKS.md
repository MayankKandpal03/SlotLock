# TASKS.md — SlotLock implementation checklist

Status: initial plan; no implementation or test results have been verified. Follow dependencies, not just the calendar. Mark `[x]` only when acceptance evidence is recorded in `PROGRESS.md`. Use `BLOCKED` in notes for unavailable prerequisites; continue independent tasks. Original plan: four weeks, 10 h/week, shared kernel ≤12 h, hosting free or ≤$10/month.

## Phase 0 — inspect and plan

- [ ] T00 — Inspect repository, branch, environment, existing files and dependencies. Record baseline, commands and blockers in PROGRESS.md. Adopt/reconcile SPEC clarification decisions D01–D10; do not silently change requirements.

## Phase 1 — foundation and auth (week 1)

- [ ] T01 — Scaffold `app/core`, `domain`, `services`, `repositories`, `api`, `infra`, `migrations`, `tests`, `k6`; create pyproject, uv lock, .gitignore, .env.example and initial README. Gate: install works; app imports; no secrets committed.
- [ ] T02 — Implement settings, SQLAlchemy session/transaction lifecycle, Redis client, request-ID middleware, structlog, problem+json errors, common Pydantic validation, keyset pagination helpers. Gate: focused tests for errors, config, request ID and pagination.
- [ ] T03 — Multi-stage non-root Dockerfile with uv; Compose API, worker, PostgreSQL, Redis; minimal worker (idempotency cleanup with documented retention). Implement `/healthz`, `/readyz`. Gate: compose config valid, fresh boot and PG/Redis readiness checks.
- [ ] T04 — Alembic migrations for users and refresh tokens; repository Protocols and SQLAlchemy adapters. Gate: fresh DB upgrade succeeds; rollback tested where safe.
- [ ] T05 — `POST /auth/register`, `/auth/login`, `/auth/refresh`, `/auth/logout`, `GET /me`; Argon2, JWT (~15 min), refresh rotation and family reuse revocation. Gate: auth tests including concurrent refresh, reused/expired token, revoked family and invalid roles.
- [ ] T06 — RBAC dependencies, service-layer identity/ownership checks; no STAFF self-registration. Gate: role tests and no privilege escalation.
- [ ] T07 — GitHub Actions CI for ruff, mypy, pytest/Testcontainers using PostgreSQL/Redis. Gate: local equivalent checks pass; CI run green if accessible. Record shared-kernel effort; no invented time logs.
- [ ] T08 (manual) — Start venue-owner conversations; record actual feedback supplied by user, not simulated as real.

## Phase 2 — venue and booking correctness (week 2)

- [ ] T09 — Venue/court/staff migrations, models, scope and APIs: `POST/GET /venues`, `GET /venues/{venue_id}`, `POST /venues/{venue_id}/courts`, `POST /venues/{venue_id}/staff`. Gate: single-venue STAFF constraint, owner/foreign authorization, validated hours, timezone and slot lengths, pagination.
- [ ] T10 — Bookings/rulesets/idempotency migrations; `btree_gist`, partial `no_overlap` exclusion constraint, range index, constraints and FK relationships. Gate: independent DB connections reject overlaps, accept adjacency and separate courts, allow rebooking after cancellation.
- [ ] T11 — Pure slot generation, venue-local dates, aligned UTC intervals and initial pricing rule support. `GET /courts/{court_id}/availability`. Gate: free/booked status, opening hours, offset handling, price per slot and boundary tests; HELD integration follows T16.
- [ ] T12 — Idempotency service: scoped advisory lock, request hash, unique key, response storage, replay, mismatch and in-flight handling. Gate: concurrent test proves atomic effect + response, 422 mismatch and 409 in-flight.
- [ ] T13 — Transactional confirmation core, channel and actor/customer attribution; map exclusion violations to 409. Integrate the public `/confirm` with T16 hold contract (do not create an insecure interim public bypass). Gate: N concurrent distinct requests at service/DB layer -> one winner, zero overlaps; idempotency and scope tested.
- [ ] T14 — `GET /bookings`, `GET /bookings/{booking_id}`, `POST /bookings/{booking_id}/cancel`; filters, cursor, ownership and immutable-rule refund quote, row lock, idempotency. Gate: cancellation releases slot, retries safe, no double refund; full cross-venue/customer authorization.
- [ ] T15 — CI correctness gate: real PG integration tests for races, overlap query, retries after lost response and cancellation; run authorization matrix for implemented routes. Gate: exact evidence in PROGRESS.md. Complete rules/hold end-to-end checks in Phase 3.

## Phase 3 — Redis, rules and operational signals (week 3)

- [ ] T16 — `POST /holds`, `DELETE /holds/{hold_id}`; five-minute atomic acquisition/reverse lookup, opaque signed hold contract per D01, quote and rule version, ownership-safe compare-and-delete. Gate: competing holds, expiry, release, stale-release successor protection, invalid/foreign holds, BOOKED precedence.
- [ ] T17 — Redis token bucket Lua on `/holds`, per-user configurable capacity/refill, Retry-After and TTL. Gate: rate limit shared across instances, refill boundaries, 429 response, no lost updates.
- [ ] T18 — Complete public `POST /confirm` with hold validation, authorization and commit/release; replay prior success even after hold expiry. Implement Redis outage path with verified signed token and fail-open rate limiter/hold, degraded readiness. Gate: real API race with distinct keys, exactly one 201 and rest 409, zero overlaps with Redis down; 410 expired, 422 mismatch, stale/forged hold denied; DB down readiness 503.
- [ ] T19 — Pure `decide(facts,ruleset)` returning outcome, reasons, rule_version; validate YAML and safe parse; sample base/peak pricing and refund tiers; initial ruleset; immutable version storage. Implement GET/POST `/venues/{venue_id}/rules` incl historical lookup. Gate: version publish concurrency, pricing/refund boundary tests and owner/staff access; existing quotes/bookings unaffected by new versions.
- [ ] T20 — Integrate rules/HELD into day-grid, hold quotes, confirm and cancel. Gate: old version used for refunds; BOOKED overrides HELD; integer money; explicit simulated refund convention; all channels same correctness path.
- [ ] T21 — `/metrics` internal access, counters for holds/confirms/409/429/Redis degradation, latency histograms, alerts on 5xx and 409 rates. Gate: scrape metrics, verify labels bounded and logs contain request IDs without secrets/PII.
- [ ] T22 — Full end-to-end regression: all endpoints, authz matrix, health/readiness, rule versions, idempotency, Redis outage, refresh reuse. Gate: ruff/mypy/pytest and Compose smoke checks; two-terminal race demo reproducible.

## Phase 4 — proof, packaging and deployment (week 4)

- [ ] T23 — k6 ES6+ scripts for availability, holds and confirm; seeded fixtures and reproducible command/config. Ensure load requests do not use production data. Gate: scripts run on test deployment; separate expected 409/429 from unexpected errors.
- [ ] T24 — Execute benchmarks in available environment; populate RESULTS.md with date, commit, machine/container specs, duration, concurrency/RPS, p50/p95/p99, status distribution, overlaps and duplicates, limitations. Gate: genuine measured numbers only. If k6 or Docker unavailable, leave measured portion blocked and provide commands.
- [ ] T25 — README setup, architecture Mermaid diagram, migrations, curl examples for all main flows, RBAC, error contract, PostgreSQL/Redis reasoning, outage/replay/cancellation semantics, demo and deployment instructions. Gate: a fresh reader can reproduce test/demo without hidden setup.
- [ ] T26 — CI/CD: main branch build and GHCR push, SSH VPS deploy configuration, health checks, rollback instructions, secret names and least privilege. Gate: workflow syntax and deployment scripts verified without exposing secrets; actual publishing/deploy only with authorization/configured infrastructure.
- [ ] T27 (external prerequisite) — Deploy to authorized VPS within budget, validate public URL, HTTPS and readiness. Record actual evidence. BLOCKED until user provides/authorizes infrastructure and deployment credentials.
- [ ] T28 (manual) — Record 2-minute demo and show three venue owners. Gate: user-supplied evidence; no invented contacts or feedback.
- [ ] T29 — Final review: compare every SPEC endpoint, schema, test, security and ops requirement against implementation; record verified and pending items, exact commands and known limitations. Derive resume bullets from actual RESULTS.md only. Distinguish implementation complete from deployed/validated complete.

## Stretch — not authorized by default

- [ ] S01 — Recurring bookings (only after explicit approval and overlap semantics defined).
- [ ] S02 — UPI advance payment link and UTR storage (only after explicit approval; define verification and refund integration).
- [ ] S03 — Thin React/TypeScript UI (only after explicit approval).

## Evidence rule

A task is not complete merely because files exist. Each checked task needs acceptance evidence (test/command output, commit or reproducible verification) in PROGRESS.md. Where the environment prevents verification, leave unchecked and record why, what was implemented and how to verify later. No mock-only proof for PostgreSQL exclusion, Redis outage or transaction correctness.
