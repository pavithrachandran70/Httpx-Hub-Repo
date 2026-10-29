# Plan — PER-81 (v3)

- made from: review feedback (architect_review)
- source: LLM planner
- complexity: moderate
- needs plan review: yes
- track hint: quick

Response cleanup ownership moved from client dispatcher to Response.aread/aiter_bytes with try/finally and shielded pool release; architect and designer rerun to propagate the new design.

## What changed

Ownership of async response cleanup moved from AsyncClient._send_single_request to Response.aread/aiter_bytes with try/finally; aclose() now shields pool release with asyncio.shield. Architect, scrum_master, and designer rerun to propagate this design change through HLD, breakdown, and LLD.

- Rerun: architect, scrum_master, designer, test_architect
- Kept (results reused): analyst

## Feedback that informed this version

- **PR comment** by pavithrachandran70 on hub#2 `[hub#2:issue-6491891340]`: HLD AD-1 and AD-4 are wrong. Do not catch CancelledError or TimeoutError in AsyncClient._send_single_request and then call aclose().

Change the design:
- Response.aread and Response.aiter_bytes own the cleanup, not the client dispatcher.
- Wrap the body read in try/finally and always call aclose() on the way out, including cancel and timeout.
- Shield the pool release inside aclose() with asyncio.shield so a second cancel during close still returns the connection.
- Update the HLD, the file touch list, and the tests to match this. The client dispatcher must not own this path.
  - addressed by: Architect rewrites AD-1 and AD-4 to place cleanup in Response.aread/aiter_bytes with try/finally and shielded pool release; designer updates file plan to reflect _models.py as primary touch, not _client.py dispatcher.

## Steps

1. [x] **analyst (requirements)** — Establish clear requirements for async response cleanup on cancel and timeout.
   - focus: Focus on the response lifecycle: who owns closing the stream, when the connection returns to the pool, and sync/async consistency guarantees.
2. [x] **architect (hld)** — Document which module owns closing the response on cancel or timeout and how the connection is returned to the pool.
   - focus: Define clear ownership boundaries: which component detects cancel/timeout, which closes the response stream, and which returns the connection to the pool. Address sync/async consistency.
3. [x] **scrum_master (breakdown)** — Convert requirements and HLD into implementable stories that designer can detail.
   - focus: Create a single story for the response cleanup fix, incorporating the architectural decisions.
4. [x] **designer (lld)** — Provide a file-level plan of the modules that change for async response close and pool release, with test coverage.
   - focus: Map out which files handle cancel detection, response cleanup, and pool return. Specify which tests cover cancel and timeout scenarios.
5. [x] **test_architect (tests)** — AC explicitly requires regression tests for cancel and timeout while streaming.
6. [x] **merge_pack (pack)** _(auto)_ — Pack per-ticket specs into merged_plan.md for review.
7. [ ] **architect_gate (review)** _(auto)_ _(rerun)_ — Human approval of specs and this plan before Dev.
8. [ ] **dev** _(auto)_ — Fixed execution step.
9. [ ] **pre_pr_qa** _(auto)_ — Fixed execution step.
10. [ ] **open_pr** _(auto)_ — Fixed execution step.
11. [ ] **pr_review** _(auto)_ — Fixed execution step.
12. [ ] **merge** _(auto)_ — Fixed execution step.

## Skipped roles

- **tpm_gate** — Scope is clear and technical; no product sign-off needed for a bug fix.
- **critic** — Linear pipeline where each role refines the previous; no parallel artifacts to cross-check.
