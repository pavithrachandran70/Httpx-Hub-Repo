---
key: PER-92
type: story
status: todo
blocks: []
blocked_by: []
labels: []
repos: []
summary: Empty / blank input validation
---

Setting a header with an empty name should raise a clear `ValueError` (or the project’s existing validation error), not silently store bad data.

Acceptance criteria

* Empty header name raises.
* Valid headers still work.
* One unit test for the empty-name case.
