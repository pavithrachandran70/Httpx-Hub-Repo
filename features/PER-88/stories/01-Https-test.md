# Story: Fix Client.stream auth inheritance (PER-88)

**ID:** PER-88  
**Type:** Bug  
**Epic:** (none — standalone backlog item)  
**Status:** draft  
**Assignee:** (unassigned)  
**Estimate:** 2 points  

---

## Summary

`Client.stream()` and `AsyncClient.stream()` do not inherit the `auth` parameter set at client instantiation, causing 401 errors when accessing authenticated endpoints unless `auth=` is passed explicitly on every `stream()` call. This is a regression from the behavior of `get()`, `post()`, and `request()`, which correctly apply client-level auth.

**Fix:** Make `stream()` methods resolve auth in the same order as other client methods: per-call `auth=` overrides client-level `auth`, which overrides no auth.

---

## Acceptance Criteria

### AC-1: stream() methods inherit client-level auth by default

**Given** a `Client` instantiated with `auth=(user, password)`  
**When** `Client.stream("GET", url)` is called without an explicit `auth=` parameter  
**Then** the stream request uses the client-level auth credentials and returns a 2xx response for an endpoint requiring those credentials

**Given** an `AsyncClient` instantiated with `auth=(user, password)`  
**When** `AsyncClient.stream("GET", url)` is called without an explicit `auth=` parameter  
**Then** the stream request uses the client-level auth credentials and returns a 2xx response for an endpoint requiring those credentials

---

### AC-2: Per-call auth parameter overrides client-level auth

**Given** a `Client` instantiated with `auth=("default_user", "default_password")`  
**When** `Client.stream("GET", url, auth=("override_user", "override_password"))` is called  
**Then** the stream request uses the override credentials, not the client-level credentials

**Given** an `AsyncClient` instantiated with `auth=("default_user", "default_password")`  
**When** `AsyncClient.stream("GET", url, auth=("override_user", "override_password"))` is called  
**Then** the stream request uses the override credentials, not the client-level credentials

---

### AC-3: stream() auth behavior matches request()/get()/post() methods

**Given** a `Client` instantiated with `auth=(user, password)`  
**When** both `Client.get(url)` and `Client.stream("GET", url)` are called against the same authenticated endpoint  
**Then** both requests succeed with the same auth credentials applied

**Given** an `AsyncClient` instantiated with `auth=(user, password)`  
**When** both `AsyncClient.get(url)` and `AsyncClient.stream("GET", url)` are called against the same authenticated endpoint  
**Then** both requests succeed with the same auth credentials applied

---

### AC-4: Regression tests verify fix for sync and async clients

**Given** the fix is implemented  
**When** the test suite runs  
**Then** new regression tests in `tests/client/test_auth.py` verify:
- Sync `Client.stream` inherits client-level auth
- Async `AsyncClient.stream` inherits client-level auth
- Per-call `auth=` override works for both sync and async `stream()`
- Auth behavior is consistent across `stream()`, `get()`, `post()`, and `request()` methods

---

## Technical Specification

### Root Cause
The `stream()` method in both `Client` and `AsyncClient` does not apply the client-level `auth` parameter when building the request. This breaks parity with `get()`, `post()`, and `request()`, which correctly resolve auth from:
1. Per-call `auth=` parameter (highest priority)
2. Client-level `auth` from constructor
3. No auth (default)

### Implementation Approach

**Files to modify:**
- Client implementation file containing `Client.stream()` method
- AsyncClient implementation file containing `AsyncClient.stream()` method

**Changes:**
1. Ensure `stream()` methods apply the same auth resolution logic as `request()`:
   - If `auth` is explicitly passed to `stream()`, use it
   - Otherwise, use `self.auth` (client-level auth)
   - Otherwise, use no auth
2. Verify this matches the existing pattern in `get()`, `post()`, and `request()` methods

**Test files to modify:**
- `tests/client/test_auth.py` — add regression tests for `stream()` auth inheritance

### Test Coverage

**New test cases in `tests/client/test_auth.py`:**

1. **test_client_stream_inherits_auth** — sync `Client.stream` uses client-level auth when no per-call `auth=` is provided
2. **test_async_client_stream_inherits_auth** — async `AsyncClient.stream` uses client-level auth when no per-call `auth=` is provided
3. **test_client_stream_auth_override** — sync `Client.stream` with per-call `auth=` overrides client-level auth
4. **test_async_client_stream_auth_override** — async `AsyncClient.stream` with per-call `auth=` overrides client-level auth
5. **test_client_stream_auth_parity_with_get** — sync `Client.stream` and `Client.get` apply the same auth credentials to the same endpoint
6. **test_async_client_stream_auth_parity_with_get** — async `AsyncClient.stream` and `AsyncClient.get` apply the same auth credentials to the same endpoint

**Test setup:**
- Mock authenticated endpoint returning 200 for correct credentials, 401 otherwise
- Use `httpx` test utilities or `respx` for mocking (per existing test patterns in `tests/client/`)

---

## Verification

### Manual Verification
Run the reproduction case from the ticket:
```python
import httpx

cli = httpx.Client(auth=("user", "password"))
cli.get(url).raise_for_status()          # Should pass

with cli.stream("GET", url) as resp:
    resp.raise_for_status()              # Should now pass (was 401 before fix)

with cli.stream("GET", url, auth=cli.auth) as resp:
    resp.raise_for_status()              # Should still pass
```

### Automated Verification
All new tests in `tests/client/test_auth.py` must pass:
```bash
pytest tests/client/test_auth.py -v
```

---

## Notes

- No API changes required; this is a behavioral fix to match documented and expected behavior
- The fix should mirror the auth resolution logic already present in `request()`, `get()`, and `post()` methods
- Both sync (`Client`) and async (`AsyncClient`) variants must be fixed
- Existing tests for `get()`, `post()`, and `request()` auth behavior should continue to pass

---

## References

- **Ticket:** PER-88
- **Repo:** Https-test (https://github.com/pavithrachandran70/httpx-demo)
- **Base branch:** master
- **Test file:** `tests/client/test_auth.py`
