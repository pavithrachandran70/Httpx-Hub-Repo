# Requirements — PER-90

**Missing export / `__all__` omission**

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

**Given** the test suite already covers exports (or a minimal import check is added)  
**When** the export test runs  
**Then** it confirms `FunctionAuth` is publicly accessible via `from httpx import FunctionAuth`

---

**Notes:**

- This is a single-line addition to `__all__` in the appropriate `httpx` module file.
- No framework, ORM, language, or cloud stack choices are made here — this is a pure export fix.
- The existing test suite file `tests/test_exported_members.py` likely already validates `__all__` coverage; confirm `FunctionAuth` appears in that check, or add a minimal import assertion if missing.
