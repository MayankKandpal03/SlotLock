# PROGRESS.md — SlotLock recovery checkpoint

This is a living document. The agent updates it during implementation; it is not a claim that development has started.

## Current state

| Field | Value |
| --- | --- |
| Project | SlotLock |
| Status | NOT VERIFIED / initial planning files prepared |
| Current phase | Phase 0 — repository inspection |
| Next task | T00 |
| Branch | Unknown — inspect Git |
| Last verified commit | None recorded |
| Working tree | Unknown — inspect before editing |
| Tests | Not run |
| CI | Not checked |
| Deployment | Not verified / requires authorization |
| Performance results | None |
| Real-user validation | None verified |

## Next action — read this first

1. Read `AGENTS.md`, `SPEC.md`, `TASKS.md` and this file.
2. Run `git status --short`, `git branch --show-current`, `git log -5 --oneline`, and inspect existing repository files. A new repo may not yet have commits; do not assume that it is empty.
3. Identify existing work and reconcile it with the checklist. Check Python/uv, Docker availability, DB/Redis connectivity and test environment. Record results below; do not expose secrets.
4. Start T00 and confirm or document the proposed D01–D10 design decisions in SPEC.md. Then complete the first unblocked Phase 1 task. Do not recreate or discard existing code.
5. After each coherent implementation slice, run checks, update this file and TASKS.md, and commit changes if permitted.

## Completed tasks

None verified. The supplied HTML scope was converted into specification, agent instructions and checklist; no application source or live environment was inspected for these planning files.

## In progress

- T00 — pending repository inspection.

## Decisions

Source-confirmed: PostgreSQL exclusion constraint is the final no-overlap guarantee; Redis holds are advisory; booking and idempotency persistence are transactional; rules are immutable/versioned; STAFF is scoped to one venue.

Proposed, not yet implemented or adopted: D01–D10 in SPEC.md. Record each decision as adopted/modified/blocked with reason and affected tests. Do not describe these as original requirements.

| ID | Decision / reason | State | Evidence |
| --- | --- | --- | --- |
| D01–D10 | See clarification register in SPEC.md | Pending review | None |

## Verification log

| Task | Command / check | Result | Commit / environment |
| --- | --- | --- | --- |
| — | — | No application checks run | — |

## Blockers and external prerequisites

| Item | Status | Next action |
| --- | --- | --- |
| Repository / dependencies | Unknown | Inspect at T00 |
| Docker / PostgreSQL / Redis / k6 | Unknown | Check tool availability and services; record exact limitations |
| GitHub Actions / GHCR | Unknown | Configure workflow; verify available permissions |
| VPS / domain / SSH secrets | Not supplied or verified | Prepare deploy config; request explicit authorization and approved infrastructure before deployment |
| Venue-owner conversations / demo | Not verified | User completes or supplies actual evidence |

## Checkpoint entry template

Append after each task or meaningful subtask, and whenever an interruption is anticipated. Keep the latest state and next action above current too.

```text
Date/time (UTC):
Task ID / phase:
Branch / base commit:
Commit or platform patch reference (if actually available):
Files changed:
What works:
What remains:
Decisions / assumptions:
Commands run and actual results:
Failed or skipped checks and why:
Migration / environment state:
Blockers and prerequisites:
Next exact step (file/function/test):
```

## Recovery after an interruption or usage limit

- Resume the existing cloud task/branch or open a new task using the most recent durable branch/patch. This document does not itself launch or schedule Codex.
- Read all four planning files. Compare recorded commit and working tree against actual state; inspect diffs and tests before redoing anything.
- If a prior slice was incomplete, preserve changes, reproduce the failure, fix the smallest relevant problem and rerun checks. Do not blindly rerun destructive migrations or production deployment.
- Re-run the last relevant passing test or document why it cannot run. Do not mark skipped tests passed.
- If previous work was never persisted, recover only from available commits/patches or explicitly report what is missing. No file can guarantee recovery from an abruptly terminated, unpersisted session.
- Continue the first unfinished unblocked task. Stop only at completion, a genuine blocker affecting all remaining work, or an imposed execution/usage limit. Checkpoint proactively rather than waiting for the limit.

## Session history

- Initial checkpoint: planning files prepared from supplied SlotLock build scope. Implementation, test, deployment and usage-reset automation not verified.
