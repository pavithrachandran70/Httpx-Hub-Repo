# Requirements — PER-92

**Empty / blank input validation**

Setting a header with an empty name should raise a clear `ValueError` (or the project's existing validation error), not silently store bad data.

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

## Technical Notes

- **Validation error type**: Use `ValueError` unless the project already has a dedicated validation error class (check existing exceptions in the codebase).
- **Test placement**: Add the test case to `tests/models/test_headers.py` alongside other header validation tests.
- **Scope**: This is a single validation check in the header-setting logic — no framework, ORM, or deployment changes.

---

## Out of Scope

- Validation of other invalid header names (e.g., whitespace-only, special characters)
- Error handling for header values
- Changes to API contracts or schema
- Cross-repo coordination
