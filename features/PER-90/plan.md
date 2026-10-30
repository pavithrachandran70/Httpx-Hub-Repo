# Plan — PER-90 (v1)

- made from: ticket
- source: LLM planner
- complexity: trivial
- needs plan review: yes
- track hint: standard

Adding a missing export to __all__ is a single-file change with clear acceptance criteria, so analyst inline_story alone produces an implementable story.

## Steps

1. [ ] **analyst (inline_story)** — The ticket states exactly what to change and where; one story with AC and implementation notes is sufficient.
   - focus: Focus on the __all__ addition and the import test; no requirements decomposition needed.
2. [ ] **architect_gate (review)** _(auto)_ — Human approval of specs and this plan before Dev.
3. [ ] **dev** _(auto)_ — Fixed execution step.
4. [ ] **pre_pr_qa** _(auto)_ — Fixed execution step.
5. [ ] **open_pr** _(auto)_ — Fixed execution step.
6. [ ] **pr_review** _(auto)_ — Fixed execution step.
7. [ ] **merge** _(auto)_ — Fixed execution step.

## Skipped roles

- **tpm_gate** — Scope is unambiguous and technical; no product sign-off needed for a missing export fix.
- **architect** — No new components, API contracts, or cross-repo concerns; adding one symbol to __all__ requires no design.
- **scrum_master** — Single-file change in one repo; no breakdown needed.
- **designer** — Implementation is trivial (add one line to __all__); no file plan or algorithm notes needed.
- **test_architect** — AC already specifies the test approach; analyst inline_story will capture it in the story.
- **critic** — Only one spec document will exist; nothing to cross-check.
