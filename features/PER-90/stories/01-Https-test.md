# Story: Add FunctionAuth to __all__ — PER-90

**Type:** story  
**Status:** draft  
**Epic:** N/A (standalone backlog ticket)  
**Assignee:** unassigned  
**Complexity:** trivial  
**Estimate:** 1 point  

---

## Description

`FunctionAuth` exists in the `httpx` codebase but is missing from `__all__`, so `from httpx import FunctionAuth` fails with `ImportError`. This story adds `FunctionAuth` to the `__all__` list in the appropriate module to make it publicly accessible.

**What changed:** One symbol added to `__all__`.  
**Why it matters:** Users expect `FunctionAuth` to be importable by name from the top-level `httpx` module.

---

## Acceptance Criteria

### AC1: FunctionAuth is exported

**Given** the `httpx` module defines `FunctionAuth`  
**When** a user writes `from httpx import FunctionAuth`  
**Then** the import succeeds without `ImportError`

### AC2: Existing imports remain unaffected

**Given** the codebase has existing import statements from `httpx`  
**When** `FunctionAuth` is added to `__all__`  
**Then** all existing imports continue to work as before

### AC3: Export is verified by test

**Given** the test suite has `tests/test_exported_members.py` covering `__all__` validation  
**When** the export test runs  
**Then** it confirms `FunctionAuth` is publicly accessible and present in the expected export list

---

## Implementation Notes (LLD)

### Files to modify

1. **`httpx/__init__.py`** (or whichever `__init__.py` holds `__all__` at the package root)  
   - Locate the `__all__` list.  
   - Add `"FunctionAuth"` in alphabetical order among the other exports.  
   - Ensure `FunctionAuth` is already imported in that file (e.g., `from ._auth import FunctionAuth` or similar). If not, add the import.

**Example change:**

```python
# httpx/__init__.py (existing structure)
from ._auth import Auth, BasicAuth, DigestAuth, FunctionAuth  # FunctionAuth already imported

__all__ = [
    "Auth",
    "BasicAuth",
    "DigestAuth",
    "FunctionAuth",  # ← ADD THIS LINE
    # ... other exports
]
```

### Test changes

**File:** `tests/test_exported_members.py`

- This file already validates that symbols in `__all__` are importable.  
- **If the test dynamically checks `__all__` membership:** No code change needed; the test will automatically pick up `FunctionAuth` once it is added.  
- **If the test uses a hardcoded expected list:** Add `"FunctionAuth"` to that list.  
- **If no export test exists (unlikely given the file name):** Add a minimal assertion:

```python
def test_function_auth_is_exported():
    from httpx import FunctionAuth
    assert FunctionAuth is not None
```

---

## Verification Plan

### Manual Check (dev)

1. Run `python -c "from httpx import FunctionAuth; print(FunctionAuth)"` — should print the class object, not raise `ImportError`.
2. Confirm existing imports in the codebase (e.g., from tests, examples) still work.

### Automated Tests

1. Run `pytest tests/test_exported_members.py` — passes.
2. Run full test suite — no regressions.

### Success Signal

- CI passes with no new failures.
- `from httpx import FunctionAuth` succeeds in a fresh Python interpreter.
- `tests/test_exported_members.py` confirms `FunctionAuth` is in `__all__` and importable.

---

## Dependencies & Sequencing

- **Prerequisites:** None (standalone fix).  
- **Blocks:** None.  
- **Parallel work allowed:** Yes.

---

## Notes

- **Repo:** `Https-test` (https://github.com/pavithrachandran70/httpx-demo)  
- **Base branch:** `master`  
- **Concrete file paths confirmed from repo tree:**  
  - `httpx/__init__.py` (inferred location of `__all__`)  
  - `tests/test_exported_members.py` (existing test file)  
- **No new files created.**  
- **No edge cases:** This is a single-line addition with deterministic behavior.  
- **Estimation basis:** Trivial — one line, one test run, under 15 minutes.

---

## Open Questions

None. The ticket specifies exactly what to add and where. The planner confirmed no decomposition is needed.

---

**End of Story — PER-90**
