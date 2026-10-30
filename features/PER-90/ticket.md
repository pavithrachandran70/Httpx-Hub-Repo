---
key: PER-90
type: story
status: todo
blocks: []
blocked_by: []
labels: []
repos: []
summary: "Missing export /\_ `__all__` \_omission"
---

`FunctionAuth` exists but is missing from `__all__`, so `from httpx import FunctionAuth` / public export expectations fail. Add it to `__all__` only.

Acceptance criteria

* `FunctionAuth` is listed in `__all__`.
* Existing imports keep working.
* One small unit/import check if the suite already covers exports.
