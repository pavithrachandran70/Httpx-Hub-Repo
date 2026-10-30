# Merged plan — PER-83

Execution order (1 ticket(s)):

1. `01`

---

## Ticket `01`

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

## Stack

- **Language:** Python 3.x
- **Async framework:** `asyncio`
- **HTTP client library:** httpx (existing codebase)
- **Test framework:** `pytest` with `pytest-asyncio`
- **Test fixtures:** Local ASGI app (async generator simulating slow streaming response)

---

## File Changes

### 1. `httpx/_client.py` (AsyncClient response cleanup)

**Purpose:** Add cancellation/timeout exception handling to close the response and return the connection to the pool.

**Change:** Wrap response body streaming in `try/except` to catch `asyncio.CancelledError` and `asyncio.TimeoutError`, call `await response.aclose()`, and re-raise the exception.

**Implementation location:** The `AsyncClient.request()` method or the layer that manages response streaming (e.g., response context manager or internal `_send_handling_redirects()`).

**Pseudo-code:**

```python
async def request(self, method, url, **kwargs):
    response = None
    try:
        # Existing request initiation and response creation
        response = await self._send_handling_redirects(...)
        # If streaming, response body consumption happens here or in user code
        return response
    except (asyncio.CancelledError, asyncio.TimeoutError):
        # Fix: close response if it exists and is not already closed
        if response is not None and not response.is_closed:
            await response.aclose()
        raise
```

**Edge cases:**
- Response is `None` (exception before response creation) → no-op
- Response already closed by user code → `aclose()` is idempotent, no-op
- Multiple concurrent requests cancelled → each handles its own response independently

**Files modified:**
- `httpx/_client.py` (add exception handling block to `AsyncClient.request()` or equivalent method)

---

### 2. `tests/client/test_async_client.py` (regression tests)

**Purpose:** Add tests covering AC1, AC2, and AC4 — cancellation and timeout during streaming read.

**New tests:**

#### Test 1: `test_async_cancel_during_stream_closes_response` (AC1)

**Setup:**
- Local ASGI app that yields bytes with `asyncio.sleep()` delays between chunks (simulate slow stream)
- `AsyncClient` with default timeout (long enough to not interfere)

**Action:**
- Wrap `client.get(url)` in `asyncio.wait_for(timeout=<short duration>)` that fires mid-stream
- `asyncio.TimeoutError` is raised during streaming read

**Assert:**
- `asyncio.TimeoutError` is raised (expected cancellation)
- No `RuntimeError` about in-flight connections on `async with client` exit
- Connection pool in-flight count is zero after client exit

**Pseudo-code:**

```python
@pytest.mark.asyncio
async def test_async_cancel_during_stream_closes_response():
    async def slow_stream_app(scope, receive, send):
        await send({
            'type': 'http.response.start',
            'status': 200,
            'headers': [[b'content-type', b'application/octet-stream']],
        })
        for _ in range(10):
            await asyncio.sleep(0.5)  # Slow stream
            await send({
                'type': 'http.response.body',
                'body': b'chunk',
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
        # Assert: no RuntimeError on exit (implicit in clean exit)
```

---

#### Test 2: `test_async_timeout_during_stream_closes_response` (AC2)

**Setup:**
- Same local ASGI app (slow stream)
- `AsyncClient(timeout=<short duration>)` that fires mid-stream

**Action:**
- Request slow-streaming endpoint
- Client timeout fires, raising timeout exception

**Assert:**
- Timeout exception is raised (httpx timeout, not `asyncio.TimeoutError` — httpx may wrap it)
- No `RuntimeError` about in-flight connections on client exit
- Connection pool in-flight count is zero

**Pseudo-code:**

```python
@pytest.mark.asyncio
async def test_async_timeout_during_stream_closes_response():
    async def slow_stream_app(scope, receive, send):
        await send({
            'type': 'http.response.start',
            'status': 200,
            'headers': [[b'content-type', b'application/octet-stream']],
        })
        for _ in range(10):
            await asyncio.sleep(0.5)
            await send({
                'type': 'http.response.body',
                'body': b'chunk',
                'more_body': True,
            })
        await send({
            'type': 'http.response.body',
            'body': b'',
            'more_body': False,
        })

    async with httpx.AsyncClient(app=slow_stream_app, timeout=1.0) as client:
        with pytest.raises(httpx.ReadTimeout):  # or httpx.TimeoutException
            await client.get("http://test/stream")
        # Assert: no RuntimeError on exit (implicit in clean exit)
```

---

#### Test 3: `test_async_cancel_during_body_iteration_closes_response` (edge case for AC1)

**Setup:**
- Local ASGI app (slow stream)
- User code explicitly iterates response body with `async for chunk in response.aiter_bytes()`

**Action:**
- Start iteration, then cancel the task mid-iteration
- Cancellation propagates during `aiter_bytes()` call

**Assert:**
- Response is closed
- No `RuntimeError` on client exit

**Pseudo-code:**

```python
@pytest.mark.asyncio
async def test_async_cancel_during_body_iteration_closes_response():
    async def slow_stream_app(scope, receive, send):
        await send({
            'type': 'http.response.start',
            'status': 200,
            'headers': [[b'content-type', b'application/octet-stream']],
        })
        for _ in range(10):
            await asyncio.sleep(0.5)
            await send({
                'type': 'http.response.body',
                'body': b'chunk',
                'more_body': True,
            })
        await send({
            'type': 'http.response.body',
            'body': b'',
            'more_body': False,
        })

    async with httpx.AsyncClient(app=slow_stream_app) as client:
        response = await client.get("http://test/stream", stream=True)
        with pytest.raises(asyncio.TimeoutError):
            async with response:
                async for chunk in asyncio.wait_for(response.aiter_bytes(), timeout=1.0):
                    pass  # Timeout mid-iteration
        # Assert: no RuntimeError on exit
```

---

### 3. `tests/client/test_client.py` (sync client parity, AC3)

**Purpose:** Verify sync `Client` exhibits the same cleanup behavior on timeout during streaming read.

**New test:**

#### Test 4: `test_sync_timeout_during_stream_closes_response` (AC3)

**Setup:**
- Local WSGI app (slow stream, using `time.sleep()` for sync delays)
- Sync `Client(timeout=<short duration>)`

**Action:**
- Request slow-streaming endpoint
- Timeout fires mid-stream

**Assert:**
- Timeout exception is raised
- No in-flight connection error on client exit (sync equivalent of `RuntimeError`)

**Pseudo-code:**

```python
def test_sync_timeout_during_stream_closes_response():
    def slow_stream_app(environ, start_response):
        status = '200 OK'
        headers = [('Content-Type', 'application/octet-stream')]
        start_response(status, headers)
        for _ in range(10):
            time.sleep(0.5)
            yield b'chunk'

    with httpx.Client(app=slow_stream_app, timeout=1.0) as client:
        with pytest.raises(httpx.ReadTimeout):
            client.get("http://test/stream")
        # Assert: no in-flight error on exit (implicit in clean exit)
```

**Note:** If sync client uses threading for timeouts, the cancellation mechanism may differ from async. Document behavior if it diverges.

---

## Connection Pooling Contract (implementation detail)

**Module:** `httpx/_transports/default.py` (or equivalent connection pool module)

**No changes required to the pool itself.** The fix leverages the existing `response.aclose()` → `pool.release(connection)` path. The pool's in-flight tracking and `RuntimeError` check remain unchanged.

**Verified behavior:**
- `response.aclose()` calls the pool's `release(connection)` method
- `release(connection)` removes the connection from the in-flight set and marks it as available
- `AsyncClient.__aexit__` (or `ConnectionPool.aclose()`) checks the in-flight set and raises `RuntimeError` if non-empty

**Post-fix invariant:** No connections remain in-flight when `AsyncClient.__aexit__()` runs, because all responses are closed by cancellation handlers or normal cleanup.

---

## Test Fixture: Local ASGI App

**Location:** `tests/client/test_async_client.py` (inline or in `conftest.py`)

**Implementation:**

```python
async def slow_stream_app(scope, receive, send):
    """ASGI app that streams bytes with delays to simulate slow response."""
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

## Risks and Mitigations

### R1: Cleanup runs during user code iteration

**Risk:** If user code is iterating the response body and cancellation occurs, the cleanup handler may attempt to close the response while iteration is in progress, causing a race or double-close.

**Mitigation:**
- `response.aclose()` is idempotent and safe to call mid-iteration
- Existing httpx response lifecycle tests cover mid-iteration close
- New test (Test 3) explicitly covers cancellation during `aiter_bytes()` to confirm no regression

**Likelihood:** Low  
**Impact:** Medium (could cause cleanup failure)  
**Status:** Accepted (mitigation via idempotent API and new test)

---

### R2: Sync client cleanup path diverges

**Risk:** Sync `Client` uses threading for timeouts; cancellation mechanism may differ from async, leading to inconsistent cleanup behavior.

**Mitigation:**
- Test 4 (`test_sync_timeout_during_stream_closes_response`) verifies sync behavior
- If sync client does not already close responses on timeout, apply the same fix (wrap streaming in `try/except` for timeout exception, call `response.close()`)
- Document any intentional behavioral differences (e.g., threading model limitations) in test comments

**Likelihood:** Medium (sync and async paths are structurally different)  
**Impact:** Medium (user confusion, duplicate bugs)  
**Status:** Accepted (mitigation via AC3 verification)

---

### R3: Performance impact of exception handling

**Risk:** Adding `try/except` blocks in the hot path of response iteration could degrade performance.

**Mitigation:**
- Exception handling in Python has negligible overhead when no exception is raised
- The fix only adds cleanup logic to the exceptional path (cancel/timeout), not the normal path
- Existing httpx response handling already includes exception handling for network errors and decoders

**Likelihood:** Very Low  
**Impact:** Low  
**Status:** Accepted (no significant overhead)

---

## Verification Steps

1. **Run new tests:** `pytest tests/client/test_async_client.py::test_async_cancel_during_stream_closes_response tests/client/test_async_client.py::test_async_timeout_during_stream_closes_response tests/client/test_async_client.py::test_async_cancel_during_body_iteration_closes_response`
   - All three tests pass
   - No `RuntimeError` about in-flight connections on client exit

2. **Run sync test:** `pytest tests/client/test_client.py::test_sync_timeout_during_stream_closes_response`
   - Test passes
   - No in-flight connection error on sync client exit

3. **Run full test suite:** `pytest tests/`
   - All existing tests pass (no regression)
   - Existing timeout tests (`tests/test_timeouts.py`) still pass

4. **Manual repro verification:** Run the reproduction case from the ticket (see `stories/01-Https-test.md`) after the fix
   - No `RuntimeError` raised on client exit
   - `asyncio.TimeoutError` is raised as expected

---

## Open Questions

(none)

---

## References

- Story: `stories/01-Https-test.md`
- HLD: `hld.md`
- Requirements: `requirements.md`
- Repo: `Https-test` (https://github.com/pavithrachandran70/httpx-demo)
- Base branch: `master`

---

# Story summaries

- `01-Https-test`: Low-Level Design — Close response stream when async request is cancelled mid-read

---

_Single-ticket pack: order is trivial (no multi-split)._
