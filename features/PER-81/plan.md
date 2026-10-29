# Plan — PER-81 (v2)

- made from: review feedback (plan_review)
- source: LLM planner
- complexity: moderate
- needs plan review: yes
- track hint: quick

This fix requires architectural clarity on ownership of response closure and connection pool release during cancel/timeout, plus a detailed file plan for the affected modules and tests.

## What changed

Added architect (HLD) and designer (LLD) to document response ownership and provide file-level implementation plan per reviewer request; added scrum_master to produce stories that designer consumes; changed analyst to requirements activity to support the pipeline.

- Added: architect, scrum_master, designer

## Feedback that informed this version

- **PR comment** by pavithrachandran70 on hub#1 `[hub#1:issue-6491577530]`: Plan change: this is not a one-file cleanup. Add the architect step before any spec work. Document which module owns closing the response on cancel or timeout, and how the connection is returned to the pool. Do not skip architect.

Also add designer. I want a file-level plan of the modules that change (async response close and pool release) and which tests cover them, before dev.
  - addressed by: Added architect step to document module ownership of response closure and pool release; added scrum_master to produce stories; added designer for file-level plan; changed analyst from inline_story to requirements to provide proper input for architect.

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

## Planner notes

- attempt 1: designer needs ['stories'] but no earlier step produces it (plan must produce stories: analyst `inline_story` or scrum_master)
