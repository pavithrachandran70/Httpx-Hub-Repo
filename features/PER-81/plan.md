# Plan — PER-81 (v1)

- made from: ticket
- source: LLM planner
- complexity: simple
- needs plan review: yes
- track hint: quick

A focused bug fix for resource cleanup on cancelled async requests; only inline story and tests are needed since the behavior is fully specified and scoped to one repo.

## Steps

1. [ ] **analyst (inline_story)** — The ticket already states clear behavior and acceptance criteria for a single-repo fix.
   - focus: Focus on the async cancellation cleanup path and ensure AC coverage includes both timeout and explicit cancel scenarios.
2. [ ] **test_architect (tests)** — AC explicitly requires regression tests for cancel and timeout scenarios.
   - focus: Map each AC point to a test case, especially the drip/stream body scenarios and the in-flight RuntimeError check.
3. [ ] **architect_gate (review)** _(auto)_ — Human approval of specs and this plan before Dev.
4. [ ] **dev** _(auto)_ — Fixed execution step.
5. [ ] **pre_pr_qa** _(auto)_ — Fixed execution step.
6. [ ] **open_pr** _(auto)_ — Fixed execution step.
7. [ ] **pr_review** _(auto)_ — Fixed execution step.
8. [ ] **merge** _(auto)_ — Fixed execution step.

## Skipped roles

- **tpm_gate** — Scope is clear and technical; no product sign-off needed for a bug fix.
- **architect** — No new components, APIs, or cross-repo changes; fix is localized to async response cleanup.
- **scrum_master** — Single story in one repo; no breakdown needed.
- **designer** — Implementation approach is straightforward from the AC; no file plan needed before coding.
- **critic** — Only one spec document produced; nothing to cross-check.
