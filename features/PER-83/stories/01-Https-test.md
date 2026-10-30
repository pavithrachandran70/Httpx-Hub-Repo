# Low-Level Design — Close response stream when async request is cancelled mid-read

**Story ID:** PER-83  
**Epic:** (none — backlog bug)  
**Status:** draft  
**Created:** 2025-01-24  
**Repo:** Https-test (https://github.com/pavithrachandran70/httpx-demo)  
**Base branch:** master

---

## Overview

This LLD specifies the file-level changes required to close the async response stream when a request is cancelled or times out mid-read, preventing the `RuntimeError: The connection pool was closed while 1 HTTP requests/responses were still in-flight.` The fix adds a `finally` block around the response streaming path in `AsyncClient`, ensuring `response.aclose()` is called before any cancellation or timeout exception propagates.

---

## Acceptance Criteria (from requirements/breakdown)

### AC1: Cancel during stream closes response and returns connection

**Given** an `AsyncClient` with a response body streaming from a slow endpoint  
**When** the request is cancelled via `asyncio.wait_for` timeout or explicit task cancellation before the stream completes  
**Then** the response is closed immediately and the connection is returned to the pool without raising an in-flight error on client exit

---

### AC2: Timeout during stream closes response and returns connection

**Given** an `AsyncClient` with a timeout configured and a response body streaming from a slow endpoint  
**When** the client timeout fires while the stream is still reading  
**Then** the response is closed immediately and the connection is returned to the pool without raising an in-flight error on client exit

---

### AC3: Sync and async cancel behavior is consistent

**Given** both sync `Client` and `AsyncClient` with streaming responses  
**When** a request is cancelled or timed out mid-read in either client  
**Then** both clients close the response and return the connection to the pool in the same manner, with no in-flight connection error on cleanup

---

### AC4: Regression tests cover cancel and timeout during streaming read

**Given** the httpx test suite  
**When** tests are run  
**Then** regression tests exist covering:
- Cancellation via `asyncio.wait_for` timeout while reading a drip/stream endpoint
- Timeout via client timeout setting while reading a drip/stream endpoint
- Verification that no in-flight `RuntimeError` is raised on client exit after cancel/timeout

---

## Stack

- **Language:** Python 3.x
- **Async framework:** `asyncio`
- **HTTP client library:** httpx (existing codebase)
- **Test framework:** `pytest` with `pytest-asyncio`
- **Test fixtures:** Local ASGI app (async generator simulating slow streaming response)

---

## File Plan

### 1. Response cleanup implementation

**File:** `httpx/_client.py`

**Change:** Add `finally` block around response streaming in `AsyncClient` methods to ensure cleanup on cancellation or timeout.

**Implementation location:** The `AsyncClient.request()` method or the layer that manages response streaming (e.g., response context manager or internal `_send_handling_redirects()`).

**Approach:**
- Wrap response body streaming in `try/finally` block
- In the `finally` block: call `await response.aclose()` if response exists and is not already closed
- Ensures cleanup runs before any `asyncio.CancelledError` or `asyncio.TimeoutError` propagates

**Interfaces:**

```python
class AsyncClient:
    async def request(self, method, url, **kwargs) -> Response:
        """
        Post-fix behavior:
        - Response streaming wrapped in try/finally block
        - Finally block calls response.aclose() before any exception propagates
        - Ensures connection is returned to pool and in-flight count is zero
          on cancellation or timeout
        """
        pass
```

**Algorithm notes:**

```python
# Pseudo-code structure (not actual implementation)
async def request(self, method, url, **kwargs):
    response = None
    try:
        # Existing request initiation and response creation
        response = await self._send_handling_redirects(...)
        # Response body consumption happens here or in user code
        return response
    finally:
        # Cleanup finally block (named: response_cleanup_on_cancel)
        # Fix: close response if it exists and is not already closed
        # This runs whether the request completes normally, is cancelled,
        # times out, or raises any other exception
        if response is not None and not response.is_closed:
            await response.aclose()
```

**Finally block naming:** The `finally` block ensuring response cleanup is explicitly named in comments as `response_cleanup_on_cancel` for clarity and traceability.

**Edge cases:**
- Response is `None` (exception before response creation) → no-op in cleanup
- Response already closed by user code → `aclose()` is idempotent, no-op
- Multiple concurrent requests cancelled → each handles its own response independently
- Exception during `aclose()` itself → propagates, but this is a separate connection pool issue

**Key invariant:** The `finally` block runs **before** any exception (cancel, timeout, or otherwise) propagates to the caller, ensuring the connection is always returned to the pool.

---

### 2. Async client regression tests

**File:** `tests/client/test_async_client.py`

**Changes:** Add three new test functions covering AC1, AC2, and AC4.

---

#### Test 1: `test_async_cancel_during_stream_closes_response` (AC1)

**Purpose:** Verify that cancelling a request via `asyncio.wait_for` timeout during streaming read closes the response and returns the connection to the pool, and that no `RuntimeError` about in-flight connections is raised on client exit.

**Setup:**
- Local ASGI app that yields bytes with `asyncio.sleep()` delays between chunks (simulate slow stream)
- `AsyncClient` with default timeout (long enough to not interfere with the test timeout)

**Action:**
- Wrap `client.get(url)` in `asyncio.wait_for(timeout=<short duration>)` that fires mid-stream
- `asyncio.TimeoutError` is raised during streaming read

**Assertions:**
- `asyncio.TimeoutError` is raised (expected cancellation)
- Connection pool in-flight count is zero after client exit (explicit check via `_inflight_requests()`)
- **AC4 guard:** No `RuntimeError` about in-flight connections on `async with client` exit (wrap exit in try/except or add explicit assertion that exit completes)

**Implementation:**

```python
@pytest.mark.asyncio
async def test_async_cancel_during_stream_closes_response():
    """
    AC1: Cancelling via asyncio.wait_for timeout during stream closes
    response and returns connection to pool.
    AC4: No RuntimeError on client exit.
    """
    async def slow_stream_app(scope, receive, send):
        await send({
            'type': 'http.response.start',
            'status': 200,
            'headers': [[b'content-type', b'application/octet-stream']],
        })
        for i in range(10):
            await asyncio.sleep(0.5)  # Slow stream
            await send({
                'type': 'http.response.body',
                'body': f'chunk{i}'.encode(),
                'more_body': True,
            })
        await send({
            'type': 'http.response.body',
            'body': b'',
            'more_body': False,
        })

    # AC4 guard: Wrap client exit to explicitly verify no RuntimeError
    try:
        async with httpx.AsyncClient(app=slow_stream_app) as client:
            with pytest.raises(asyncio.TimeoutError):
                await asyncio.wait_for(client.get("http://test/stream"), timeout=1.0)
            # Verify connection returned to pool before __aexit__
            assert _inflight_requests(client) == []
        # Explicit AC4 verification: __aexit__ completed without RuntimeError
    except RuntimeError as e:
        if "connection pool was closed while" in str(e):
            pytest.fail(f"AC4 violation: {e}")
        raise
```

---

#### Test 2: `test_async_timeout_during_stream_closes_response` (AC2)

**Purpose:** Verify that a client-configured timeout during streaming read closes the response and returns the connection to the pool, and that no `RuntimeError` about in-flight connections is raised on client exit.

**Setup:**
- Same local ASGI app (slow stream)
- `AsyncClient(timeout=<short duration>)` that fires mid-stream

**Action:**
- Request slow-streaming endpoint
- Client timeout fires, raising timeout exception (httpx may raise `httpx.ReadTimeout` or `httpx.TimeoutException`)

**Assertions:**
- Timeout exception is raised (expected)
- Connection pool in-flight count is zero
- **AC4 guard:** No `RuntimeError` about in-flight connections on client exit (wrap exit in try/except or add explicit assertion)

**Implementation:**

```python
@pytest.mark.asyncio
async def test_async_timeout_during_stream_closes_response():
    """
    AC2: Client-configured timeout during stream closes response and
    returns connection to pool.
    AC4: No RuntimeError on client exit.
    """
    async def slow_stream_app(scope, receive, send):
        await send({
            'type': 'http.response.start',
            'status': 200,
            'headers': [[b'content-type', b'application/octet-stream']],
        })
        for i in range(10):
            await asyncio.sleep(0.5)
            await send({
                'type': 'http.response.body',
                'body': f'chunk{i}'.encode(),
                'more_body': True,
            })
        await send({
            'type': 'http.response.body',
            'body': b'',
            'more_body': False,
        })

    # AC4 guard: Wrap client exit to explicitly verify no RuntimeError
    try:
        async with httpx.AsyncClient(app=slow_stream_app, timeout=1.0) as client:
            with pytest.raises((httpx.ReadTimeout, httpx.TimeoutException)):
                await client.get("http://test/stream")
            # Verify connection returned to pool before __aexit__
            assert _inflight_requests(client) == []
        # Explicit AC4 verification: __aexit__ completed without RuntimeError
    except RuntimeError as e:
        if "connection pool was closed while" in str(e):
            pytest.fail(f"AC4 violation: {e}")
        raise
```

---

#### Test 3: `test_async_cancel_during_body_iteration_closes_response` (AC1 edge case)

**Purpose:** Verify that cancellation during explicit body iteration (user code calling `response.aiter_bytes()`) closes the response and returns the connection to the pool, and that no `RuntimeError` about in-flight connections is raised on client exit.

**Setup:**
- Local ASGI app (slow stream)
- User code explicitly iterates response body with `async for chunk in response.aiter_bytes()`

**Action:**
- Start iteration, then cancel the task mid-iteration via `asyncio.wait_for`
- Cancellation propagates during `aiter_bytes()` call

**Assertions:**
- Response is closed
- Connection pool in-flight count is zero
- **AC4 guard:** No `RuntimeError` on client exit

**Implementation:**

```python
@pytest.mark.asyncio
async def test_async_cancel_during_body_iteration_closes_response():
    """
    AC1 edge case: Cancellation during explicit body iteration closes
    response and returns connection to pool.
    AC4: No RuntimeError on client exit.
    """
    async def slow_stream_app(scope, receive, send):
        await send({
            'type': 'http.response.start',
            'status': 200,
            'headers': [[b'content-type', b'application/octet-stream']],
        })
        for i in range(10):
            await asyncio.sleep(0.5)
            await send({
                'type': 'http.response.body',
                'body': f'chunk{i}'.encode(),
                'more_body': True,
            })
        await send({
            'type': 'http.response.body',
            'body': b'',
            'more_body': False,
        })

    # AC4 guard: Wrap client exit to explicitly verify no RuntimeError
    try:
        async with httpx.AsyncClient(app=slow_stream_app) as client:
            response = await client.get("http://test/stream", stream=True)
            
            async def consume_with_timeout():
                async with response:
                    async for chunk in response.aiter_bytes():
                        pass  # Will timeout mid-iteration
            
            with pytest.raises(asyncio.TimeoutError):
                await asyncio.wait_for(consume_with_timeout(), timeout=1.0)
            
            # Verify connection returned to pool before __aexit__
            assert _inflight_requests(client) == []
        # Explicit AC4 verification: __aexit__ completed without RuntimeError
    except RuntimeError as e:
        if "connection pool was closed while" in str(e):
            pytest.fail(f"AC4 violation: {e}")
        raise
```

---

### 3. Sync client parity test

**File:** `tests/client/test_client.py`

**Change:** Add one new test function covering AC3 (sync client parity).

---

#### Test 4: `test_sync_timeout_during_stream_closes_response` (AC3)

**Purpose:** Verify sync `Client` exhibits the same cleanup behavior on timeout during streaming read as the async client: closes the response, returns the connection to the pool, and exits without raising an in-flight connection `RuntimeError`.

**Setup:**
- Local WSGI app (slow stream, using `time.sleep()` for sync delays)
- Sync `Client(timeout=<short duration>)`

**Action:**
- Request slow-streaming endpoint
- Timeout fires mid-stream

**Assertions:**
- Timeout exception is raised
- Connection pool in-flight count is zero
- **AC3 guard:** No in-flight connection `RuntimeError` on client exit (wrap exit in try/except or add explicit assertion)

**Implementation:**

```python
import time

def test_sync_timeout_during_stream_closes_response():
    """
    AC3: Sync client timeout during stream closes response and returns
    connection to pool, consistent with async behavior.
    No RuntimeError on client exit.
    """
    def slow_stream_app(environ, start_response):
        status = '200 OK'
        headers = [('Content-Type', 'application/octet-stream')]
        start_response(status, headers)
        for i in range(10):
            time.sleep(0.5)
            yield f'chunk{i}'.encode()

    # AC3 guard: Wrap client exit to explicitly verify no RuntimeError
    try:
        with httpx.Client(app=slow_stream_app, timeout=1.0) as client:
            with pytest.raises((httpx.ReadTimeout, httpx.TimeoutException)):
                client.get("http://test/stream")
            # Verify connection returned to pool before __exit__
            # (sync equivalent check, if pool exposes _requests or similar)
            pool = client._transport._pool
            assert list(pool._requests) == []
        # Explicit AC3 verification: __exit__ completed without RuntimeError
    except RuntimeError as e:
        if "connection pool was closed while" in str(e):
            pytest.fail(f"AC3 violation: {e}")
        raise
```

**Note:** If sync client uses threading for timeouts, the cancellation mechanism may differ structurally from async. The `finally` block cleanup pattern applies the same way: `response.close()` in the `finally` block around streaming. Document behavior differences if found during implementation.

---

## Test Fixture: Local ASGI App

**Location:** Inline in `tests/client/test_async_client.py` (or move to `conftest.py` if reused widely)

**Purpose:** Simulate slow-streaming response for testing cancellation/timeout behavior without external dependencies.

**Implementation:**

```python
async def slow_stream_app(scope, receive, send):
    """
    ASGI app that streams bytes with delays to simulate slow response.
    Used by cancellation/timeout tests to trigger mid-stream exceptions.
    """
    await send({
        'type': 'http.response.start',
        'status': 200,
        'headers': [[b'content-type', b'application/octet-stream']],
    })
    for i in range(10):
        await asyncio.sleep(0.5)  # Simulate slow stream
        await send({
            'type': 'http.response.body',
            'body': f'chunk{i}'.encode(),
            'more_body': True,
        })
    await send({
        'type': 'http.response.body',
        'body': b'',
        'more_body': False,
    })
```

**Usage:** Pass as `app=slow_stream_app` to `AsyncClient()` constructor in tests.

---

## Connection Pooling Contract (unchanged)

**Module:** `httpx/_transports/default.py` (or equivalent connection pool implementation)

**No changes required to the pool itself.** The fix leverages the existing `response.aclose()` → `pool.release(connection)` path.

**Verified behavior:**
- `response.aclose()` calls the pool's `release(connection)` method
- `release(connection)` removes the connection from the in-flight set and marks it as available
- `AsyncClient.__aexit__` (or `ConnectionPool.aclose()`) checks the in-flight set and raises `RuntimeError` if non-empty

**Post-fix invariant:** No connections remain in-flight when `AsyncClient.__aexit__()` runs, because all responses are closed by the `finally` block before any exception propagates.

---

## Algorithm Notes

### Response cleanup on cancellation (finally block pattern)

**Pattern:**

```python
# Named finally block: response_cleanup_on_cancel
response = None
try:
    # Create response and initiate streaming
    response = await self._send_handling_redirects(...)
    return response
finally:
    # response_cleanup_on_cancel: Close response before any exception propagates
    # This runs on normal completion, cancellation, timeout, or any other exception
    if response is not None and not response.is_closed:
        await response.aclose()
```

**Key invariants:**
- `finally` block runs **before** any exception propagates (cancel, timeout, or otherwise)
- Cleanup is unconditional: response is closed whether the request completes normally or abnormally
- `response.is_closed` check avoids double-close (idempotent, but explicit check is clearer)
- Normal return flow also benefits: response is closed even if user does not consume the body

**Why `finally` over `except`:**
- `finally` runs on **all** exit paths (normal return, exception, cancellation)
- `except (CancelledError, TimeoutError)` would miss other exceptions (e.g., network errors)
- `finally` ensures cleanup runs before any exception propagates, preventing in-flight connection errors

### Connection pool interaction

**Sequence (with fix):**

1. `AsyncClient.request()` calls `pool.acquire()` → connection moves to in-flight set
2. Response object created, streaming begins
3. **Cancellation/timeout exception raised during streaming** (or normal completion)
4. **Fix:** `finally` block (`response_cleanup_on_cancel`) runs, calls `await response.aclose()`
5. `response.aclose()` calls `pool.release(connection)` → connection moves to available set, removed from in-flight set
6. Exception propagates to user code (if any), or normal return proceeds
7. `AsyncClient.__aexit__` calls `pool.aclose()` → checks in-flight set, finds zero connections, clean exit

---

## Tests

### Test Coverage Summary (4 tests total)

**File:** `tests/client/test_async_client.py` (3 tests)
1. `test_async_cancel_during_stream_closes_response` — AC1 (asyncio.wait_for cancel) + AC4 (no RuntimeError on exit)
2. `test_async_timeout_during_stream_closes_response` — AC2 (client timeout) + AC4 (no RuntimeError on exit)
3. `test_async_cancel_during_body_iteration_closes_response` — AC1 edge case (user iteration cancel) + AC4 (no RuntimeError on exit)

**File:** `tests/client/test_client.py` (1 test)
4. `test_sync_timeout_during_stream_closes_response` — AC3 (sync client parity, no RuntimeError on exit)

### Verification Approach (4 items, matching breakdown count)

1. **Run new async tests:** `pytest tests/client/test_async_client.py::test_async_cancel_during_stream_closes_response tests/client/test_async_client.py::test_async_timeout_during_stream_closes_response tests/client/test_async_client.py::test_async_cancel_during_body_iteration_closes_response`
   - All three tests pass
   - No `RuntimeError` about in-flight connections on client exit in any test (explicitly verified by try/except wrapper)

2. **Run sync test:** `pytest tests/client/test_client.py::test_sync_timeout_during_stream_closes_response`
   - Test passes
   - No in-flight connection error on sync client exit (explicitly verified by try/except wrapper)

3. **Run full test suite:** `pytest tests/`
   - All existing tests pass (no regression)
   - Existing timeout tests (`tests/test_timeouts.py`) still pass

4. **Manual repro verification:** Run the reproduction case from the ticket after the fix
   - No `RuntimeError` raised on client exit
   - `asyncio.TimeoutError` is raised as expected

---

## Risks and Mitigations

### R1: Cleanup runs during user code iteration

**Risk:** If user code is iterating the response body and cancellation occurs, the `finally` block may attempt to close the response while iteration is in progress, causing a race or double-close.

**Mitigation:**
- `response.aclose()` is idempotent and safe to call mid-iteration (existing httpx contract)
- Existing httpx response lifecycle tests cover mid-iteration close scenarios
- New test (Test 3: `test_async_cancel_during_body_iteration_closes_response`) explicitly covers cancellation during `aiter_bytes()` to confirm no regression

**Likelihood:** Low (response API is designed for this)  
**Impact:** Medium (could cause cleanup failure if not handled)  
**Status:** Accepted (mitigation via idempotent API and new test coverage)

---

### R2: Sync client cleanup path diverges

**Risk:** Sync `Client` uses threading for timeouts; cancellation mechanism may differ from async, leading to inconsistent cleanup behavior.

**Mitigation:**
- Test 4 (`test_sync_timeout_during_stream_closes_response`) verifies sync behavior
- If sync client does not already close responses on timeout, apply the same fix (wrap streaming in `try/finally`, call `response.close()` in the `finally` block)
- Document any intentional behavioral differences (e.g., threading model limitations) in test comments or implementation notes

**Likelihood:** Medium (sync and async code paths are structurally different)  
**Impact:** Medium (user confusion, duplicate bugs)  
**Status:** Accepted (mitigation via AC3 verification and test coverage)

---

### R3: Performance impact of finally block

**Risk:** Adding a `finally` block in the hot path of response iteration could degrade performance for the common case (no cancellation).

**Mitigation:**
- `finally` blocks in Python have negligible overhead when no exception is raised
- The `finally` block only adds a single conditional check (`if response is not None and not response.is_closed`) in the normal path
- The `response.aclose()` call in the `finally` block runs **in all cases** (normal or exceptional), but this is consistent with existing httpx response lifecycle management (responses should always be closed)
- Existing httpx response handling already includes similar cleanup logic in context managers

**Likelihood:** Very Low (finally blocks and conditional checks are cheap in Python)  
**Impact:** Low (performance is not materially affected)  
**Status:** Accepted (no significant overhead)

---

## Open Questions

(none)

---

## References

- Story: `stories/01-Https-test.md`
- HLD: `hld.md`
- Requirements: `requirements.md`
- Ticket: PER-83
- Repo: `Https-test` (https://github.com/pavithrachandran70/httpx-demo)
- Base branch: `master`
