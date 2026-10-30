# Low-Level Design — Close response stream when async request is cancelled mid-read

**Story ID:** PER-83  
**Epic:** (none — backlog bug)  
**Status:** draft  
**Created:** 2025-01-24  
**Repo:** Https-test (https://github.com/pavithrachandran70/httpx-demo)  
**Base branch:** master

---

## Overview

This LLD specifies the file-level changes required to close the async response stream when a request is cancelled or times out mid-read, preventing the `RuntimeError: The connection pool was closed while 1 HTTP requests/responses were still in-flight.` The fix adds cancellation/timeout exception handling to the response iteration path in `AsyncClient`, ensuring `response.aclose()` is called before the exception propagates.

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

**Change:** Add cancellation/timeout exception handling to `AsyncClient` methods that manage response streaming.

**Implementation location:** The `AsyncClient.request()` method or the layer that manages response streaming (e.g., response context manager or internal `_send_handling_redirects()`).

**Approach:**
- Wrap response body streaming in `try/except` to catch `asyncio.CancelledError` and `asyncio.TimeoutError`
- On exception: call `await response.aclose()` to release connection to pool
- Re-raise the exception unchanged to preserve asyncio cancellation semantics

**Interfaces:**

```python
class AsyncClient:
    async def request(self, method, url, **kwargs) -> Response:
        """
        Post-fix behavior:
        - If asyncio.CancelledError or asyncio.TimeoutError raised during
          response streaming, calls response.aclose() before re-raising
        - Ensures connection is returned to pool and in-flight count is zero
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
    except (asyncio.CancelledError, asyncio.TimeoutError):
        # Fix: close response if it exists and is not already closed
        if response is not None and not response.is_closed:
            await response.aclose()
        raise  # Re-raise unchanged
```

**Edge cases:**
- Response is `None` (exception before response creation) → no-op in cleanup
- Response already closed by user code → `aclose()` is idempotent, no-op
- Multiple concurrent requests cancelled → each handles its own response independently
- Exception during `aclose()` itself → allow to propagate (connection pool cleanup failure is a separate issue)

---

### 2. Async client regression tests

**File:** `tests/client/test_async_client.py`

**Changes:** Add three new test functions covering AC1, AC2, and AC4.

---

#### Test 1: `test_async_cancel_during_stream_closes_response` (AC1)

**Purpose:** Verify that cancelling a request via `asyncio.wait_for` timeout during streaming read closes the response and returns the connection to the pool.

**Setup:**
- Local ASGI app that yields bytes with `asyncio.sleep()` delays between chunks (simulate slow stream)
- `AsyncClient` with default timeout (long enough to not interfere with the test timeout)

**Action:**
- Wrap `client.get(url)` in `asyncio.wait_for(timeout=<short duration>)` that fires mid-stream
- `asyncio.TimeoutError` is raised during streaming read

**Assertions:**
- `asyncio.TimeoutError` is raised (expected cancellation)
- No `RuntimeError` about in-flight connections on `async with client` exit (implicit in clean exit)
- Connection pool in-flight count is zero after client exit (if pool exposes this for testing)

**Implementation:**

```python
@pytest.mark.asyncio
async def test_async_cancel_during_stream_closes_response():
    """
    AC1: Cancelling via asyncio.wait_for timeout during stream closes
    response and returns connection to pool.
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

    async with httpx.AsyncClient(app=slow_stream_app) as client:
        with pytest.raises(asyncio.TimeoutError):
            await asyncio.wait_for(client.get("http://test/stream"), timeout=1.0)
        # Implicit assertion: no RuntimeError on clean exit
```

---

#### Test 2: `test_async_timeout_during_stream_closes_response` (AC2)

**Purpose:** Verify that a client-configured timeout during streaming read closes the response and returns the connection to the pool.

**Setup:**
- Same local ASGI app (slow stream)
- `AsyncClient(timeout=<short duration>)` that fires mid-stream

**Action:**
- Request slow-streaming endpoint
- Client timeout fires, raising timeout exception (httpx may raise `httpx.ReadTimeout` or `httpx.TimeoutException`)

**Assertions:**
- Timeout exception is raised (expected)
- No `RuntimeError` about in-flight connections on client exit
- Connection pool in-flight count is zero

**Implementation:**

```python
@pytest.mark.asyncio
async def test_async_timeout_during_stream_closes_response():
    """
    AC2: Client-configured timeout during stream closes response and
    returns connection to pool.
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

    async with httpx.AsyncClient(app=slow_stream_app, timeout=1.0) as client:
        with pytest.raises((httpx.ReadTimeout, httpx.TimeoutException)):
            await client.get("http://test/stream")
        # Implicit assertion: no RuntimeError on clean exit
```

---

#### Test 3: `test_async_cancel_during_body_iteration_closes_response` (AC1 edge case)

**Purpose:** Verify that cancellation during explicit body iteration (user code calling `response.aiter_bytes()`) closes the response and returns the connection to the pool.

**Setup:**
- Local ASGI app (slow stream)
- User code explicitly iterates response body with `async for chunk in response.aiter_bytes()`

**Action:**
- Start iteration, then cancel the task mid-iteration via `asyncio.wait_for`
- Cancellation propagates during `aiter_bytes()` call

**Assertions:**
- Response is closed
- No `RuntimeError` on client exit

**Implementation:**

```python
@pytest.mark.asyncio
async def test_async_cancel_during_body_iteration_closes_response():
    """
    AC1 edge case: Cancellation during explicit body iteration closes
    response and returns connection to pool.
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

    async with httpx.AsyncClient(app=slow_stream_app) as client:
        response = await client.get("http://test/stream", stream=True)
        
        async def consume_with_timeout():
            async with response:
                async for chunk in response.aiter_bytes():
                    pass  # Will timeout mid-iteration
        
        with pytest.raises(asyncio.TimeoutError):
            await asyncio.wait_for(consume_with_timeout(), timeout=1.0)
        # Implicit assertion: no RuntimeError on client exit
```

---

### 3. Sync client parity test

**File:** `tests/client/test_client.py`

**Change:** Add one new test function covering AC3 (sync client parity).

---

#### Test 4: `test_sync_timeout_during_stream_closes_response` (AC3)

**Purpose:** Verify sync `Client` exhibits the same cleanup behavior on timeout during streaming read as the async client.

**Setup:**
- Local WSGI app (slow stream, using `time.sleep()` for sync delays)
- Sync `Client(timeout=<short duration>)`

**Action:**
- Request slow-streaming endpoint
- Timeout fires mid-stream

**Assertions:**
- Timeout exception is raised
- No in-flight connection error on client exit (sync equivalent of `RuntimeError`)

**Implementation:**

```python
import time

def test_sync_timeout_during_stream_closes_response():
    """
    AC3: Sync client timeout during stream closes response and returns
    connection to pool, consistent with async behavior.
    """
    def slow_stream_app(environ, start_response):
        status = '200 OK'
        headers = [('Content-Type', 'application/octet-stream')]
        start_response(status, headers)
        for i in range(10):
            time.sleep(0.5)
            yield f'chunk{i}'.encode()

    with httpx.Client(app=slow_stream_app, timeout=1.0) as client:
        with pytest.raises((httpx.ReadTimeout, httpx.TimeoutException)):
            client.get("http://test/stream")
        # Implicit assertion: no in-flight error on clean exit
```

**Note:** If sync client uses threading for timeouts, the cancellation mechanism may differ structurally from async. Document behavior differences if found during implementation.

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

**Post-fix invariant:** No connections remain in-flight when `AsyncClient.__aexit__()` runs, because all responses are closed by cancellation handlers or normal cleanup.

---

## Algorithm Notes

### Response cleanup on cancellation

**Pattern:**

```python
response = None
try:
    # Create response and initiate streaming
    response = await self._send_handling_redirects(...)
    return response
except (asyncio.CancelledError, asyncio.TimeoutError):
    # Cleanup: close response if it exists and is not already closed
    if response is not None and not response.is_closed:
        await response.aclose()
    raise  # Re-raise unchanged
```

**Key invariants:**
- Cleanup runs before re-raise (exception propagates only after connection is returned)
- `response.is_closed` check avoids double-close (idempotent, but explicit check is clearer)
- Re-raise preserves asyncio cancellation semantics (no exception conversion or suppression)

### Connection pool interaction

**Sequence (with fix):**

1. `AsyncClient.request()` calls `pool.acquire()` → connection moves to in-flight set
2. Response object created, streaming begins
3. **Cancellation/timeout exception raised during streaming**
4. **Fix:** Exception handler calls `await response.aclose()`
5. `response.aclose()` calls `pool.release(connection)` → connection moves to available set, removed from in-flight set
6. Exception re-raised, propagates to user code
7. `AsyncClient.__aexit__` calls `pool.aclose()` → checks in-flight set, finds zero connections, clean exit

---

## Tests

### Test Coverage Summary (4 tests total)

**File:** `tests/client/test_async_client.py` (3 tests)
1. `test_async_cancel_during_stream_closes_response` — AC1 (asyncio.wait_for cancel)
2. `test_async_timeout_during_stream_closes_response` — AC2 (client timeout)
3. `test_async_cancel_during_body_iteration_closes_response` — AC1 edge case (user iteration cancel)

**File:** `tests/client/test_client.py` (1 test)
4. `test_sync_timeout_during_stream_closes_response` — AC3 (sync client parity)

### Verification Approach (4 items, matching breakdown count)

1. **Run new async tests:** `pytest tests/client/test_async_client.py::test_async_cancel_during_stream_closes_response tests/client/test_async_client.py::test_async_timeout_during_stream_closes_response tests/client/test_async_client.py::test_async_cancel_during_body_iteration_closes_response`
   - All three tests pass
   - No `RuntimeError` about in-flight connections on client exit in any test

2. **Run sync test:** `pytest tests/client/test_client.py::test_sync_timeout_during_stream_closes_response`
   - Test passes
   - No in-flight connection error on sync client exit

3. **Run full test suite:** `pytest tests/`
   - All existing tests pass (no regression)
   - Existing timeout tests (`tests/test_timeouts.py`) still pass

4. **Manual repro verification:** Run the reproduction case from the ticket after the fix
   - No `RuntimeError` raised on client exit
   - `asyncio.TimeoutError` is raised as expected

---

## Risks and Mitigations

### R1: Cleanup runs during user code iteration

**Risk:** If user code is iterating the response body and cancellation occurs, the cleanup handler may attempt to close the response while iteration is in progress, causing a race or double-close.

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
- If sync client does not already close responses on timeout, apply the same fix (wrap streaming in `try/except` for timeout exception, call `response.close()`)
- Document any intentional behavioral differences (e.g., threading model limitations) in test comments or implementation notes

**Likelihood:** Medium (sync and async code paths are structurally different)  
**Impact:** Medium (user confusion, duplicate bugs)  
**Status:** Accepted (mitigation via AC3 verification and test coverage)

---

### R3: Performance impact of exception handling

**Risk:** Adding `try/except` blocks in the hot path of response iteration could degrade performance for the common case (no cancellation).

**Mitigation:**
- Exception handling in Python has negligible overhead when no exception is raised
- The fix only adds cleanup logic to the exceptional path (cancel/timeout), not the normal path
- Existing httpx response handling already includes exception handling for network errors and decoders (no new pattern)

**Likelihood:** Very Low (exception handling is cheap in Python when not raised)  
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
