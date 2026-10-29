# LLD — Story 01-Https-test: Implement response cleanup on async cancel/timeout with sync parity and tests

**Ticket ID:** PER-82  
**Story:** Implement response cleanup on async cancel/timeout with sync parity and tests  
**Type:** Bug  
**Repo:** Https-test  
**Date:** 2025-01-23

---

## Acceptance Criteria

*(Copied verbatim from requirements and breakdown)*

- **AC-1:** Cancelling or timing out a streaming read always closes the response and returns the connection to the pool.
- **AC-2:** Closing `AsyncClient` after cancel does not raise the in-flight RuntimeError.
- **AC-3:** Sync and async behaviour stay consistent for the same cancel path.
- **AC-4:** Add regression tests for cancel and timeout while reading a drip/stream body.

---

## File Plan

### Implementation files (httpx source — not in repo tree)

**Note:** The httpx library source is not present in the provided repo tree (`/home/pavc/Documents/Https-Demo`). The implementer must ensure httpx is available in editable mode or cloned separately. Paths below are inferred from standard httpx structure and must be verified/located at implementation time.

#### 1. **`httpx/_client.py`** (async request dispatcher — primary fix)

**Location inference:** httpx source root → `httpx/_client.py` or `httpx/_async_client.py`  
**Purpose:** Add exception handling around response body read in async request flow.  
**Changes:**
- Locate the async request method (likely `AsyncClient.request()` or `AsyncClient._send_single_request()`) that orchestrates the HTTP request and consumes the response body.
- Wrap the response body consumption (e.g., the section that awaits `response.aiter_bytes()`, `response.aread()`, or the internal method that reads the stream) in a `try/except` block.
- Catch `asyncio.CancelledError` and `asyncio.TimeoutError`.
- In the exception handler, call `await response.aclose()` before re-raising the exception.
- Verify no conflict with existing `try/finally` or `async with response` context manager cleanup.

**Pseudocode (conceptual):**
```python
# In AsyncClient.request() or _send_single_request()
try:
    # existing response body read logic
    # e.g., await response.aread() or for chunk in response.aiter_bytes(): ...
    ...
except (asyncio.CancelledError, asyncio.TimeoutError):
    await response.aclose()
    raise
```

**Expected line delta:** +10-15 lines (try/except block, cleanup call)

#### 2. **`httpx/_client.py`** (sync request dispatcher — parity fix, if separate)

**Location inference:** httpx source root → `httpx/_client.py` (sync `Client` class, if distinct from async)  
**Purpose:** Apply parallel exception handling for sync client to match async behavior.  
**Changes:**
- Locate sync `Client.request()` or equivalent method.
- If sync client has separate streaming read logic, wrap in `try/except` for sync interrupt exceptions (e.g., `KeyboardInterrupt`, or whatever httpx uses for sync timeout).
- Call `response.close()` before re-raising.
- If sync and async share the same dispatcher (unlikely but possible), this step may be N/A.

**Pseudocode (conceptual):**
```python
# In sync Client.request()
try:
    # existing response body read logic
    ...
except <SyncInterruptException>:  # e.g., KeyboardInterrupt, or httpx's sync timeout exception
    response.close()
    raise
```

**Expected line delta:** +10-15 lines (or 0 if sync client does not exist separately)

#### 3. **`httpx/_models.py`** (Response class — audit and potential shielding)

**Location inference:** httpx source root → `httpx/_models.py`  
**Purpose:** Verify `Response.aclose()` and `Response.close()` are idempotent and cancel-safe; add shielding if pool return is not protected.  
**Audit steps:**
- Locate `Response.aclose()` (async) and `Response.close()` (sync) methods.
- Check for idempotency guard (e.g., `if self._closed: return` at the top of the method).
- Trace pool return call (likely `self._connection.return_to_pool()` or `self._pool.release(self._connection)` or similar).
- Verify pool return is wrapped in `asyncio.shield()` (async) or `try/finally` (both) to ensure it completes even if `aclose()` is cancelled.

**Potential changes (if not shielded):**
```python
async def aclose(self) -> None:
    if self._closed:
        return
    self._closed = True
    # ... stream close logic ...
    # Shield pool return from cancellation
    await asyncio.shield(self._pool.release(self._connection))
```

**Expected line delta:** 0-20 lines (0 if already correct, +5-10 for shielding if needed, +5 for idempotency guard if missing)

#### 4. **`httpx/_async_pool.py`** (connection pool — audit only)

**Location inference:** httpx source root → `httpx/_async_pool.py` or `httpx/_pool.py`  
**Purpose:** Verify pool's connection release logic decrements in-flight count correctly when `response.aclose()` calls it.  
**Audit steps:**
- Locate pool's `release()` or `return_connection()` method.
- Confirm it decrements in-flight counter (e.g., `self._in_flight -= 1` or similar).
- Confirm pool does **not** detect cancel/timeout itself (per AD-4); it only receives release signals from `response.aclose()`.

**Expected changes:** None (audit only; document findings in Notes section)

---

### Test files (created in repo tree)

#### 5. **`tests/client/test_async_client_cancel.py`** (new file)

**Purpose:** Regression tests for async cancel and timeout scenarios.  
**Tests to implement:**

**Test 1: `test_cancel_during_streaming_read`**
- **Setup:** Start `AsyncClient`, initiate GET to slow endpoint (`https://httpbin.org/drip?duration=5` or mock equivalent).
- **Action:** Wrap call in `asyncio.wait_for(client.get(url), timeout=3)` (timeout < response duration).
- **Assert:** `asyncio.TimeoutError` is raised; `async with client` context exit does not raise `RuntimeError` about in-flight connections.
- **Verifies:** AC-1, AC-2

**Test 2: `test_client_timeout_during_streaming_read`**
- **Setup:** `AsyncClient(timeout=3)`, GET to slow endpoint (duration > 3s).
- **Action:** Let client's internal timeout fire (not `asyncio.wait_for`).
- **Assert:** `httpx.TimeoutException` (or httpx's timeout error) is raised; client exits cleanly.
- **Verifies:** AC-1, AC-2

**Test 3: `test_response_closed_after_cancel`**
- **Setup:** Start streaming read, cancel via `asyncio.wait_for`.
- **Action:** Catch exception, inspect `response.is_closed` (or equivalent attribute).
- **Assert:** `response.is_closed == True`.
- **Verifies:** AC-1 (response closure)

**Test 4: `test_connection_returned_to_pool_after_cancel`**
- **Setup:** Start streaming read, cancel mid-stream.
- **Action:** After cancel, inspect pool state (if `client._pool` exposes `._connections` or `.in_flight_count`).
- **Assert:** In-flight count is 0 (or connection is marked as available).
- **Verifies:** AC-1 (pool return)
- **Note:** If pool internals are not exposed, this test is satisfied by absence of `RuntimeError` in Test 1.

**Expected line count:** ~100-120 lines (4 required tests, setup/teardown, fixtures)

**Test fixture/server:**
- Check `tests/conftest.py` or `tests/common.py` for existing slow-response fixtures.
- If `httpbin.org` is not available in CI, use a pytest fixture with `httpx.MockTransport` or `pytest-httpserver` to simulate a slow/drip endpoint.
- Document fixture setup in inline comments.

#### 6. **`tests/client/test_client_cancel.py`** (new file, if sync client exists)

**Purpose:** Mirror async tests for sync `Client` to verify AC-3 (sync/async parity).  
**Conditional creation:** Only if sync `Client` exists as a separate implementation with its own streaming read logic.  
**Tests to implement:** Same as `test_async_client_cancel.py`, adapted for sync:

**Test 1: `test_cancel_during_streaming_read` (sync)**
- Use `threading.Timer` to simulate interrupt, or `signal` to send `SIGALRM` (platform-dependent).
- Replace `asyncio.wait_for` with sync timeout mechanism.
- Assert `TimeoutError` (or `KeyboardInterrupt`) is raised; `with client` exits cleanly.

**Test 2: `test_client_timeout_during_streaming_read` (sync)**
- `Client(timeout=3)`, GET to slow endpoint.
- Assert timeout error raised, clean exit.

**Test 3: `test_response_closed_after_cancel` (sync)**
- Interrupt during sync read, assert `response.is_closed == True`.

**Test 4: `test_connection_returned_to_pool_after_cancel` (sync)**
- Interrupt, verify pool state.

**Expected line count:** ~100-120 lines (or 0 if sync client does not exist)

**Determination of sync client existence:**
- During implementation, check `httpx/_client.py` for a `Client` class with `request()` method.
- If `Client` is only a thin wrapper around `AsyncClient` (e.g., running async methods in a thread), sync tests may not apply.
- Document findings in Notes section; if sync client does not separately implement streaming, mark AC-3 as satisfied by non-divergence and skip sync tests.

---

### Audit-only files (read, no expected changes)

#### 7. **`tests/client/test_async_client.py`** (existing)

**Purpose:** Verify no existing tests conflict with new cleanup logic (Risk 4).  
**Audit steps:**
- Search for existing tests that use `async with response` or call `response.aclose()` explicitly.
- Check for tests that mock or stub response lifecycle (e.g., tests that expect `aclose()` to be called exactly once).
- Ensure new `try/except` cleanup does not break assumptions in existing tests.

**Expected changes:** None (audit only; note any conflicts in Open Questions or Risks if found)

#### 8. **`httpx/_transports/` or `httpx/_backends/`** (if exists)

**Purpose:** Confirm cancel/timeout is **not** detected at transport or backend layer (per AD-4).  
**Audit steps:**
- Locate transport/backend modules (e.g., `httpx/_transports/default.py`, `httpx/_backends/asyncio.py`).
- Search for exception handling around send/receive operations.
- Verify transport does **not** catch `CancelledError` or attempt to close the response.

**Expected changes:** None (audit only; document if transport handles cancel differently than expected)

---

## Interfaces

### 1. Response cleanup interface (behavioral contract)

**Existing method (no signature change):**
```python
class Response:
    async def aclose(self) -> None:
        """
        Close the response stream and return the connection to the pool.
        Idempotent: safe to call multiple times.
        Cancel-safe: pool return is shielded from cancellation (after fix).
        """
        ...
```

**Caller:** `AsyncClient._send_single_request()` (or equivalent dispatcher) on `CancelledError`/`TimeoutError`.  
**Guarantee (post-fix):**
- Calling `aclose()` closes the underlying stream.
- Calling `aclose()` returns the connection to the pool atomically.
- Calling `aclose()` multiple times is a no-op after the first call (idempotent).
- Pool return completes even if `aclose()` is cancelled mid-execution (shielded).

**Sync equivalent:**
```python
class Response:
    def close(self) -> None:
        """Sync version of aclose(). Same guarantees."""
        ...
```

---

### 2. Request dispatcher exception handling (new behavior)

**Async dispatcher (conceptual interface):**
```python
class AsyncClient:
    async def request(self, method: str, url: str, **kwargs) -> Response:
        """
        Sends HTTP request and returns response.
        Post-fix: catches CancelledError/TimeoutError during response read,
        calls response.aclose(), re-raises exception.
        """
        response = await self._send_single_request(...)
        try:
            # consume response body (if stream=False) or return response (if stream=True)
            ...
        except (asyncio.CancelledError, asyncio.TimeoutError):
            await response.aclose()
            raise
        return response
```

**Sync dispatcher (if separate):**
```python
class Client:
    def request(self, method: str, url: str, **kwargs) -> Response:
        """
        Sync version. Catches sync interrupt exceptions, calls response.close(), re-raises.
        """
        response = self._send_single_request(...)
        try:
            # consume response body or return response
            ...
        except <SyncInterruptException>:
            response.close()
            raise
        return response
```

**Notes:**
- The exact location of the `try/except` block depends on whether `request()` itself consumes the body or delegates to a helper.
- If httpx uses `stream=True` mode by default and the user consumes the body later, the `try/except` may need to be in a context manager or iterator.
- LLD assumes the dispatcher is responsible for the cleanup, per AD-1 and AD-4; adjust if code inspection shows a different pattern.

---

## Algorithm Notes

### Async cleanup flow (step-by-step)

1. **User code:** `await asyncio.wait_for(client.get(url), timeout=3)`
2. **AsyncClient.get()** → calls `request("GET", url)`.
3. **AsyncClient.request()** → calls internal dispatcher (e.g., `_send_single_request()`).
4. **Dispatcher:** Sends HTTP request, creates `Response` object, starts streaming response body.
5. **Dispatcher:** Enters `try/except` block around response body read.
6. **Response body read:** Awaits chunks from `response.aiter_bytes()` (or equivalent).
7. **Timeout fires:** `asyncio.wait_for` raises `asyncio.TimeoutError` (or `CancelledError` if explicitly cancelled).
8. **Exception handler:** Catches `TimeoutError`/`CancelledError`, calls `await response.aclose()`.
9. **response.aclose():** Closes stream, calls `await asyncio.shield(pool.release(connection))` (shielded to ensure completion).
10. **Pool.release():** Decrements in-flight count, marks connection as available.
11. **Exception re-raised:** Propagates `TimeoutError` back to user code.
12. **AsyncClient context exit:** `async with client` → `__aexit__` checks pool state → no in-flight connections → clean exit (no `RuntimeError`).

### Sync cleanup flow (parallel to async)

1. **User code:** `with client: response = client.get(url)` (or threading-based timeout).
2. **Sync interrupt:** `KeyboardInterrupt` or timeout exception raised during response read.
3. **Exception handler in `Client.request()`:** Catches interrupt, calls `response.close()`.
4. **response.close():** Closes stream, calls `pool.release(connection)` (sync, no shield needed but wrapped in `try/finally`).
5. **Pool.release():** Decrements in-flight count.
6. **Exception re-raised:** Propagates to user code.
7. **Client context exit:** `with client` → `__exit__` checks pool → clean exit.

### Idempotency guarantee (response.aclose())

**Pseudocode:**
```python
async def aclose(self):
    if self._closed:
        return  # no-op if already closed
    self._closed = True
    await self._stream.aclose()  # close underlying stream
    await asyncio.shield(self._pool.release(self._connection))  # shielded pool return
```

**Key points:**
- `_closed` flag prevents double-close logic.
- `asyncio.shield()` ensures pool return completes even if `aclose()` is cancelled after the flag is set.
- First call closes stream and returns connection; subsequent calls return immediately.

### Cancel-safety guarantee (shielding pool return)

**Scenario:** `aclose()` is called in exception handler, but another cancel fires during `aclose()` execution (e.g., outer timeout).  
**Without shield:** `pool.release(connection)` might be interrupted, leaving connection in-flight.  
**With shield:** `asyncio.shield(pool.release(...))` wraps the pool return, ensuring it runs to completion even if outer cancel fires.

**Pseudocode:**
```python
await asyncio.shield(self._pool.release(self._connection))
```

**Note:** If the current httpx implementation does not shield pool return, this is a bug (Risk 2). The fix adds the shield.

---

## Tests

### Test suite structure

**File:** `tests/client/test_async_client_cancel.py`  
**Test count:** 4 required tests (matching breakdown verification approach count)  
**File (conditional):** `tests/client/test_client_cancel.py`  
**Test count (if sync applies):** 4 mirroring async = **4 sync tests**  
**Total test count:** **4 async tests** (8 total if sync client exists and requires separate tests)

### Test 1: `test_cancel_during_streaming_read` (async)

**Verifies:** AC-1 (response closure on cancel), AC-2 (no RuntimeError on client exit)

**Setup:**
```python
import asyncio
import httpx
import pytest

@pytest.mark.asyncio
async def test_cancel_during_streaming_read():
    # Use httpbin.org/drip or mock fixture
    url = "https://httpbin.org/drip?duration=5"
    async with httpx.AsyncClient() as client:
        with pytest.raises(asyncio.TimeoutError):
            await asyncio.wait_for(client.get(url), timeout=2)
    # Assert: no RuntimeError raised during context exit (implicit)
```

**Key assertion:** No `RuntimeError: The connection pool was closed while 1 HTTP requests/responses were still in-flight.` is raised when exiting `async with client`.

**Alternative (if httpbin not available):** Use `pytest-httpserver` or `httpx.MockTransport` to simulate slow response:
```python
async def slow_handler(request):
    await asyncio.sleep(5)
    return httpx.Response(200, text="slow")

transport = httpx.MockTransport(slow_handler)
async with httpx.AsyncClient(transport=transport) as client:
    with pytest.raises(asyncio.TimeoutError):
        await asyncio.wait_for(client.get("https://test.local"), timeout=2)
```

### Test 2: `test_client_timeout_during_streaming_read` (async)

**Verifies:** AC-2 (client timeout fires, clean exit)

**Setup:**
```python
@pytest.mark.asyncio
async def test_client_timeout_during_streaming_read():
    url = "https://httpbin.org/drip?duration=5"
    async with httpx.AsyncClient(timeout=2.0) as client:
        with pytest.raises(httpx.TimeoutException):  # or httpx.ReadTimeout
            await client.get(url)
    # No RuntimeError on exit
```

**Key assertion:** `httpx.TimeoutException` raised, client exits cleanly.

### Test 3: `test_response_closed_after_cancel` (async)

**Verifies:** AC-1 (response is actually closed after cancel)

**Setup:**
```python
@pytest.mark.asyncio
async def test_response_closed_after_cancel():
    url = "https://httpbin.org/drip?duration=5"
    async with httpx.AsyncClient() as client:
        response = None
        try:
            response = await asyncio.wait_for(client.get(url, stream=True), timeout=1)
        except asyncio.TimeoutError:
            pass
        assert response is not None
        assert response.is_closed  # or response.is_stream_consumed, depending on httpx API
```

**Key assertion:** `response.is_closed == True` after cancel.

**Note:** If httpx `Response` does not expose `is_closed` attribute, check for `response._content is None` or inspect internal state; adapt based on actual httpx API.

### Test 4: `test_connection_returned_to_pool_after_cancel` (async)

**Verifies:** AC-1 (connection pool state is clean after cancel)

**Setup:**
```python
@pytest.mark.asyncio
async def test_connection_returned_to_pool_after_cancel():
    url = "https://httpbin.org/drip?duration=5"
    async with httpx.AsyncClient() as client:
        # Inspect pool state before (if accessible)
        # pool = client._pool (if exposed)
        try:
            await asyncio.wait_for(client.get(url), timeout=1)
        except asyncio.TimeoutError:
            pass
        # Assert pool in-flight count is 0 (if pool exposes this)
        # If not exposed, rely on absence of RuntimeError as proxy
```

**Key assertion:** If `client._pool` or pool internals are accessible, assert `pool._num_in_flight == 0` (or equivalent). If not accessible, this test is implicitly verified by Test 1 (no `RuntimeError` on exit).

**Fallback:** If pool state is not inspectable, document this test as "implicitly verified by Test 1" and skip explicit assertion.

---

### Sync tests (if applicable)

**File:** `tests/client/test_client_cancel.py`

**Test 1 (sync): `test_cancel_during_streaming_read`**
```python
import threading
import httpx
import pytest

def test_cancel_during_streaming_read():
    url = "https://httpbin.org/drip?duration=5"
    def timeout_interrupt():
        # Send interrupt signal or raise exception in thread
        ...
    timer = threading.Timer(2, timeout_interrupt)
    timer.start()
    with httpx.Client() as client:
        with pytest.raises(TimeoutError):  # or KeyboardInterrupt
            client.get(url)
    timer.cancel()
    # No RuntimeError on exit
```

**Note:** Sync timeout/cancel mechanism depends on how httpx implements sync timeouts. Check httpx docs; adapt test accordingly.

**Tests 2-4 (sync):** Mirror async tests 2-4, replacing `async with` with `with`, `await` with blocking calls, and `asyncio.TimeoutError` with sync equivalents.

---

## Notes

### httpx source location resolution

- **Problem:** httpx source code is not in the provided repo tree (`/home/pavc/Documents/Https-Demo` contains only `tests/` directory).
- **Resolution:** Implementer must ensure httpx is available:
  - **Option 1 (editable install):** Clone httpx repo, install in editable mode (`pip install -e /path/to/httpx`), modify in place.
  - **Option 2 (site-packages):** Modify httpx in `site-packages/httpx/` (not recommended for upstream contribution; good for local testing).
  - **Option 3 (submodule):** If httpx is a Git submodule of the test repo, locate it via `git submodule status`.
- **Action:** At implementation start, run `python -c "import httpx; print(httpx.__file__)"` to find installed location. Document actual paths in implementation notes.

### Sync client existence determination

- **Check:** Open `httpx/_client.py` (or equivalent) and search for `class Client` (non-async).
- **If found:** Verify `Client.request()` has separate streaming logic (not just a wrapper around `AsyncClient` in a thread). If separate, apply sync fix and tests.
- **If not found or deprecated:** Document "Sync client does not exist or shares async implementation; AC-3 is satisfied by non-divergence." Skip sync tests.
- **Current assumption:** httpx **does** have a sync `Client`; implementer must confirm.

### Test fixture for slow endpoint

- **Default:** Use `https://httpbin.org/drip?duration=5` for tests (requires internet access in CI).
- **Fallback:** If httpbin unavailable, use `pytest-httpserver` or `httpx.MockTransport`:
  ```python
  @pytest.fixture
  def slow_server(httpserver):
      httpserver.expect_request("/slow").respond_with_data("slow", mimetype="text/plain", headers={"Content-Length": "4"})
      return httpserver
  
  async def slow_handler(request):
      await asyncio.sleep(5)
      return httpx.Response(200, text="slow")
  ```
- **Action:** Check `tests/conftest.py` for existing fixtures; prefer existing setup over adding new dependencies.

### Existing cleanup hooks audit

- **Audit `Response.__aexit__` (async context manager):**
  - Verify it calls `self.aclose()`.
  - Ensure new `try/except` cleanup in dispatcher does not conflict (should not, since both call the same `aclose()` method which is idempotent).
- **Audit existing `try/finally` in dispatcher:**
  - Search `httpx/_client.py` for `try/finally` around response operations.
  - Document any existing cleanup; ensure new exception handler integrates (e.g., existing `finally` may already call `aclose()` on normal path; new `except` adds cancel path).

### Open question resolutions

**Q1 (shield in aclose):** Resolved by step 3 of file plan (audit `httpx/_models.py`, add shield if missing).  
**Q2 (cancel detection point):** Resolved by step 1 of file plan (locate dispatcher, confirm it is the correct layer).  
**Q3 (existing cleanup hooks):** Resolved by audit in step 1 (check for `try/finally`, `__aexit__`).  
**Q4 (sync client existence):** Resolved by inspection at implementation start (document findings).  
**Q5 (test server fixture):** Resolved by checking `tests/conftest.py` (document choice in test file comments).

---

## Implementation Checklist

- [ ] Locate httpx source files (`_client.py`, `_models.py`, `_async_pool.py`); document actual paths.
- [ ] Implement async cleanup in `httpx/_client.py` (or `_async_client.py`): add `try/except (CancelledError, TimeoutError)`, call `await response.aclose()`, re-raise.
- [ ] Verify sync client exists; if yes, implement parallel cleanup in sync `Client.request()`.
- [ ] Audit `Response.aclose()` in `httpx/_models.py`: verify idempotency, add shield around pool return if missing.
- [ ] Audit `httpx/_async_pool.py`: verify release logic, confirm no cancel detection in pool.
- [ ] Create `tests/client/test_async_client_cancel.py`: implement Tests 1-4 (required).
- [ ] Create `tests/client/test_client_cancel.py`: implement sync mirror tests if sync client exists.
- [ ] Run manual repro test (reproduction case from ticket description); verify no `RuntimeError` after fix.
- [ ] Run `pytest tests/client/test_async_client_cancel.py`; verify all tests pass.
- [ ] Run `pytest tests/client/test_client_cancel.py` (if applicable); verify all tests pass.
- [ ] Run existing test suite (`pytest tests/client/test_async_client.py`); verify no regressions.
- [ ] Document any deviations from plan (e.g., actual file paths, sync client N/A) in PR description or inline comments.

---

**End of LLD.**
