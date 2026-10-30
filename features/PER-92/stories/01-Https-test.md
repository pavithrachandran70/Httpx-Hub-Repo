# Story — Empty / blank input validation

**ID:** PER-92  
**Type:** story  
**Status:** draft  
**Epic:** _(none — standalone backlog story)_  
**Assignee:** _(unassigned)_

---

## Description

Setting a header with an empty name should raise a clear `ValueError` (or the project's existing validation error), not silently store bad data.

This is a single validation fix in the header-setting logic. The validation should occur when a header name is set to an empty string (`""`), ensuring the system rejects invalid input early.

---

## Acceptance Criteria

### AC1: Empty header name raises

**Given** the header-setting interface  
**When** a header is set with an empty name (`""`)  
**Then** a `ValueError` is raised with a message indicating the header name cannot be empty

### AC2: Valid headers still work

**Given** the header-setting interface  
**When** a header is set with a non-empty name (e.g., `"Content-Type"`)  
**Then** the header is stored successfully without raising an error

### AC3: One unit test for the empty-name case

**Given** the test suite in `tests/models/test_headers.py`  
**When** tests are run  
**Then** there exists a test verifying that setting a header with an empty name raises `ValueError`

---

## Technical Specification

### Implementation Location

- **Validation logic**: The header-setting method in the headers model (likely in the main source tree, not shown in the repo-tree but inferred from test paths).
- **Test file**: `tests/models/test_headers.py`

### Validation Approach

1. **Check for empty string**: When a header name is set, validate that `name` is not an empty string.
2. **Error type**: Raise `ValueError` (unless the codebase has a dedicated validation exception — check existing patterns in the source).
3. **Error message**: Clear, actionable message: `"Header name cannot be empty"` or similar.

### Test Specification

**Test function**: Add to `tests/models/test_headers.py`

```python
def test_empty_header_name_raises_error():
    """Setting a header with an empty name should raise ValueError."""
    # Arrange: instantiate headers object (pattern from existing tests)
    
    # Act & Assert: attempt to set header with empty name
    with pytest.raises(ValueError, match="Header name cannot be empty"):
        headers[""] = "some-value"
```

**Verification approach**: 
- The test uses `pytest.raises` to assert that `ValueError` is raised.
- The `match` parameter ensures the error message is clear.
- Follow the pattern established in existing tests in `tests/models/test_headers.py` for instantiating the headers object.

### Edge Cases Covered

- Empty string (`""`) as header name → raises `ValueError` (AC1)
- Non-empty string (e.g., `"Content-Type"`) → works normally (AC2)

### Edge Cases Explicitly Out of Scope

- Whitespace-only header names (e.g., `"   "`)
- Special characters or invalid patterns in header names
- Validation of header values
- Case sensitivity of header names

---

## Implementation Notes

- **Repo**: `Https-test`
- **Base branch**: `master`
- **Remote**: https://github.com/pavithrachandran70/httpx-demo
- **Complexity**: Trivial — single validation check, single test case
- **No framework changes**: Pure validation logic in existing model
- **No schema/API changes**: Internal validation only

---

## Verification Plan

1. **Unit test passes**: `test_empty_header_name_raises_error` in `tests/models/test_headers.py` passes.
2. **Existing tests pass**: No regression in other header tests.
3. **Manual check**: Attempting to set a header with an empty name in a REPL or interactive session raises the expected error.

---

## Out of Scope

- Validation of other invalid header names (whitespace-only, special characters)
- Error handling for header values
- Changes to API contracts or schema
- Cross-repo coordination
- Performance optimization of header validation

---

## Open Questions

_(None — scope is clear from ticket ACs)_

---

## References

- Ticket: PER-92
- Test file: `tests/models/test_headers.py`
- Plan: `story-per-92-plan.md`
