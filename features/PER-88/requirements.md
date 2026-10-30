# Requirements — Client.stream ignores auth from Client constructor

**Ticket:** PER-88  
**Type:** Bug  
**Epic:** (none — standalone backlog item)

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
**Then** new regression tests in `tests/client/test_auth.py` (or equivalent) verify:
- Sync `Client.stream` inherits client-level auth
- Async `AsyncClient.stream` inherits client-level auth
- Per-call `auth=` override works for both sync and async `stream()`
- Auth behavior is consistent across `stream()`, `get()`, `post()`, and `request()` methods

---

## Technical Notes

### Current Behavior (Bug)
- `Client.stream()` and `AsyncClient.stream()` do not apply the `auth` parameter set at client instantiation
- Workaround requires passing `auth=client.auth` on every `stream()` call
- Other methods (`get`, `post`, `request`) correctly inherit client-level auth

### Expected Behavior (Fix)
- `stream()` methods resolve auth in the same order as other client methods:
  1. Use per-call `auth=` if provided
  2. Otherwise, use client-level `auth` from constructor
  3. Otherwise, use no auth
- No API surface changes required; this is a behavioral fix

### Verification Approach
- Unit tests confirm auth resolution order for `stream()` matches `get()`/`post()`
- Integration test with a mocked auth-required endpoint verifies the fix end-to-end
- Tests cover both sync (`Client`) and async (`AsyncClient`) variants

---

## Out of Scope

- Changes to auth parameter API or signature
- Auth behaviors unrelated to client-level vs per-call resolution
- Performance optimization of auth handling
- Documentation updates (handled separately if needed)
