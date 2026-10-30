# Plan — PER-83 (v4)

- made from: review feedback (architect_review)
- source: LLM planner
- complexity: simple
- needs plan review: yes
- track hint: quick

Single behavioral fix in one repo with clear scope; rerunning designer only to add finally block around stream read in AsyncClient.

## What changed

Rerunning designer only to add finally block specification around stream read in AsyncClient for proper cleanup on cancel and timeout.

- Rerun: designer
- Kept (results reused): analyst, architect

## Feedback that informed this version

- **PR comment** by pavithrachandran70 on hub#5 `[hub#5:issue-6516499740]`: Spec change: rerun designer only. Keep analyst and architect. Do not add or remove any role.

In the LLD, the cleanup must call response.aclose() from a finally block around the stream read, so cancel and timeout both close the response before the error propagates. Name that finally block in the file plan for AsyncClient, and keep the regression tests in tests/client/test_async_client.py.
  - addressed by: Designer rerun with instructions to specify response.aclose() in a finally block around stream read in AsyncClient file plan, keeping tests in test_async_client.py

## Steps

1. [x] **analyst (inline_story)** — The ticket already specifies the bug, acceptance criteria, and a repro; one implementable story is sufficient.
2. [x] **architect (hld)** — Document which module closes the async response on cancel or timeout and how the connection is returned to the pool.
3. [x] **designer (lld)** — Write the file-level plan of the modules that change and the tests that cover them.
   - focus: In the AsyncClient file plan, specify that response.aclose() must be called from a finally block around the stream read, ensuring both cancel and timeout close the response before the error propagates. Name the finally block explicitly in the file plan for httpx/_client.py. Keep the regression tests in tests/client/test_async_client.py.
4. [x] **merge_pack (pack)** _(auto)_ — Pack per-ticket specs into merged_plan.md for review.
5. [x] **architect_gate (review)** _(auto)_ — Human approval of specs and this plan before Dev.
6. [ ] **dev** _(auto)_ — Fixed execution step.
7. [x] **pre_pr_qa** _(auto)_ — Fixed execution step.
8. [x] **open_pr** _(auto)_ — Fixed execution step.
9. [ ] **pr_review** _(auto)_ — Fixed execution step.
10. [ ] **merge** _(auto)_ — Fixed execution step.

## Skipped roles

- **tpm_gate** — Scope is clear and internal: fixing a resource leak bug with no ambiguity.
- **scrum_master** — Single behavioral fix in one repo does not need breakdown into multiple stories.
- **test_architect** — Designer will write the test plan as part of the file-level design; no separate test architecture step needed.
- **critic** — Only three sequential spec documents (story, HLD, LLD) with no conflicting concerns to cross-check.
