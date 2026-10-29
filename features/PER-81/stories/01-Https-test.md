# LLD — Story 01-Https-test: Implement response cleanup on async cancel/timeout with sync parity and tests

**Ticket ID:** PER-82  
**Story:** Implement response cleanup on async cancel/timeout with sync parity and tests  
**Type:** Bug  
**Repo:** Https-test  
**Altitude:** Story  
**Date:** 2025-01-23

---

## Acceptance Criteria

### AC-1: Response closure on cancel

**Given** an `AsyncClient` with an active streaming response  
**When** the request is cancelled via `asyncio.wait_for` timeout or explicit cancellation  
**Then** the response stream is closed and the connection is returned to the pool  
**And** no in-flight connection error is raised when the client exits

### AC-2: Response closure on timeout

**Given** an `AsyncClient` with an active streaming response  
**When** the request times out via the client's configured timeout  
**Then** the response stream is closed and the connection is returned to the pool  
**And** no in-flight connection error is raised when the client exits

### AC-3: Sync/async consistency

**Given** equivalent cancel or timeout scenarios in sync and async clients  
**When** a streaming read is interrupted  
**Then** both sync `Client` and `AsyncClient` exhibit the same cleanup behavior  
**And** both properly close the response and return the connection to the pool

### AC-4: Regression test coverage

**Given** the fix is implemented  
**When** regression tests are executed  
**Then** tests exist that verify cancel and timeout during streaming reads  
**And** tests confirm no `RuntimeError` about in-flight connections when client closes  
**And** tests cover both `asyncio.wait_for` cancellation and client-timeout paths

---

## File Plan

### Implementation Files (httpx source)

**Critical context:** The httpx library source is **not present** in the provided repo tree (`/home/pavc/Documents/Https-Demo`). The implementer must locate httpx source files before proceeding:

```bash
# Find httpx installation
python -c "import httpx; print(httpx.__file__)"
# Expected output: /path/to/site-packages/httpx/__init__.py
# Source files will be in /path/to/site-packages/httpx/
```

All paths below are relative to the httpx source root (not the test repo root).

#### 1. `httpx/_models.py` (Response class — primary changes)

**Location:** `httpx/_models.py` in httpx source directory  
**Purpose:** Add try/finally cleanup to async/sync response streaming methods; shield pool release in `aclose()`

**Changes:**

**Part A: `Response.aread()` — add try/finally wrapper**

Current signature (no changes):
```python
async def aread(self) -> bytes:
    """Read and return the entire response body."""
```

**Add try/finally around body read operation:**
```python
async def aread(self) -> bytes:
    """Read and return the entire response body."""
    try:
        # Existing body read logic
        # (likely: accumulate chunks from self.aiter_bytes() or consume self._stream)
        body = b""
        async for chunk in self.aiter_bytes():
            body += chunk
        return body
    finally:
        # Always close response, even on cancel/timeout
        await self.aclose()
```

**Implementation notes:**
- Locate existing body-read logic (may be more complex than simple accumulation).
- Preserve all existing logic inside `try` block.
- Add `finally: await self.aclose()` at same indentation as `try`.
- Do **not** catch exceptions; let `CancelledError`/`TimeoutError` propagate after cleanup.

**Part B: `Response.aiter_bytes()` — add try/finally wrapper**

Current signature (no changes):
```python
async def aiter_bytes(self, chunk_size: int = None) -> AsyncIterator[bytes]:
    """Iterate over the response body as chunks of bytes."""
```

**Add try/finally around iteration logic:**
```python
async def aiter_bytes(self, chunk_size: int = None) -> AsyncIterator[bytes]:
    """Iterate over the response body as chunks of bytes."""
    try:
        # Existing iteration logic
        # (likely: async for chunk in self._stream, with optional chunk_size handling)
        async for chunk in self._stream:
            yield chunk
    finally:
        # Always close response, even on cancel/timeout
        await self.aclose()
```

**Implementation notes:**
- Generator function; `finally` executes when generator is closed (including on cancel).
- Locate existing stream iteration (may involve `self._stream.aiter_bytes(chunk_size)` or similar).
- Preserve all existing logic inside `try` block.
- Add `finally: await self.aclose()` after iteration completes or is cancelled.

**Part C: `Response.aclose()` — shield pool release**

Current signature (no changes):
```python
async def aclose(self) -> None:
    """Close the response and release the connection back to the pool."""
```

**Add idempotency guard and shield pool release:**
```python
async def aclose(self) -> None:
    """Close the response and release the connection back to the pool."""
    # Idempotency guard (may already exist)
    if self.is_closed:
        return
    
    # Mark as closed
    self._is_closed = True
    
    # Close underlying stream (existing logic, preserve as-is)
    if self._stream is not None:
        await self._stream.aclose()
    
    # Shield pool release from cancellation
    # (locate existing pool return call, wrap in asyncio.shield)
    if self._connection is not None:
        import asyncio
        # Original call (example): await self._pool.release(self._connection)
        # Shielded version:
        await asyncio.shield(self._pool.release(self._connection))
```

**Implementation notes:**
- Audit existing `aclose()` for idempotency guard (`if self.is_closed: return` or similar).
- If no guard exists, add one at the top (check `self.is_closed` or `self._is_closed` attribute).
- Locate the pool release call (syntax may vary: `await self._pool.release(...)`, `await self._connection.aclose()`, etc.).
- Wrap **only** the pool release operation in `asyncio.shield()`, not the entire method.
- Import `asyncio` at top of method or module if not already imported.

**Expected line delta:** +25-35 lines across all three async methods

---

**Part D: Sync equivalents — `Response.read()` and `Response.iter_bytes()`**

**Only if sync methods exist with separate logic** (audit required; see Open Questions).

**`Response.read()` (sync):**
```python
def read(self) -> bytes:
    """Read and return the entire response body."""
    try:
        # Existing body read logic (sync)
        body = b""
        for chunk in self.iter_bytes():
            body += chunk
        return body
    finally:
        # Always close response
        self.close()
```

**`Response.iter_bytes()` (sync):**
```python
def iter_bytes(self, chunk_size: int = None) -> Iterator[bytes]:
    """Iterate over the response body as chunks of bytes."""
    try:
        # Existing iteration logic (sync)
        for chunk in self._stream:
            yield chunk
    finally:
        # Always close response
        self.close()
```

**`Response.close()` (sync) — verify idempotency:**
- Audit existing `close()` implementation.
- Ensure it has idempotency guard (e.g., `if self.is_closed: return`).
- Pool release in sync is typically non-cancellable (no need for shield), but wrap in try/finally if not already present.

**Expected line delta (if sync exists):** +15-25 lines

---

#### 2. `httpx/_async_pool.py` (audit only, no changes expected)

**Location:** `httpx/_async_pool.py` or `httpx/_pool.py` in httpx source  
**Purpose:** Verify pool's `release()` method correctly handles connection return

**Audit steps:**
1. Locate `async def release(self, connection)` or equivalent method.
2. Confirm it decrements in-flight connection count (e.g., `self._connections_in_use -= 1` or similar).
3. Confirm pool does **not** detect `CancelledError`/`TimeoutError` itself (should only react to `release()` calls).
4. Document findings in implementation notes.

**Expected changes:** None (audit only)

---

### Test Files (created in repo tree)

#### 3. `tests/client/test_async_client_cancel.py` (new file)

**Location:** `tests/client/test_async_client_cancel.py` in repo root (`/home/pavc/Documents/Https-Demo`)  
**Purpose:** Regression tests for async cancel and timeout scenarios

**File structure:**
```python
"""
Regression tests for response cleanup on async cancel and timeout.

Verifies AC-1, AC-2, AC-4.
"""

import asyncio
import pytest
import httpx


@pytest.mark.asyncio
async def test_cancel_during_streaming_read():
    """
    Verify cancel during streaming closes response and returns connection to pool.
    
    AC-1: Cancel via asyncio.wait_for timeout should close response.
    AC-2: AsyncClient context exit should not raise RuntimeError.
    """
    url = "https://httpbin.org/drip?duration=5"
    
    async with httpx.AsyncClient() as client:
        with pytest.raises(asyncio.TimeoutError):
            # Timeout fires mid-stream (duration=5s, timeout=2s)
            await asyncio.wait_for(client.get(url), timeout=2.0)
    
    # Implicit assertion: no RuntimeError raised during client context exit


@pytest.mark.asyncio
async def test_client_timeout_during_streaming_read():
    """
    Verify client timeout during streaming closes response cleanly.
    
    AC-1: Client timeout should close response.
    AC-2: AsyncClient context exit should not raise RuntimeError.
    """
    url = "https://httpbin.org/drip?duration=5"
    
    async with httpx.AsyncClient(timeout=2.0) as client:
        with pytest.raises(httpx.TimeoutException):
            # Client timeout fires mid-stream (duration=5s, client timeout=2s)
            await client.get(url)
    
    # Implicit assertion: no RuntimeError on exit


@pytest.mark.asyncio
async def test_response_closed_after_cancel():
    """
    Verify response.is_closed is True after cancel.
    
    AC-1: Response must be closed after cancel, not left in half-open state.
    """
    url = "https://httpbin.org/drip?duration=5"
    response = None
    
    async with httpx.AsyncClient() as client:
        try:
            # Start streaming response
            response = await client.get(url, stream=True)
            # Cancel during body read
            await asyncio.wait_for(response.aread(), timeout=1.0)
        except asyncio.TimeoutError:
            pass  # Expected
    
    # Verify response was closed by aread's finally block
    assert response is not None, "Response should have been created"
    assert response.is_closed, "Response should be closed after cancel"


@pytest.mark.asyncio
async def test_connection_returned_to_pool_after_cancel():
    """
    Verify connection pool state is clean after cancel.
    
    AC-1: Connection must be returned to pool, not left in-flight.
    
    Note: This test may require pool internals access. If pool does not expose
    in-flight count, the absence of RuntimeError in other tests verifies this AC.
    """
    url = "https://httpbin.org/drip?duration=5"
    
    async with httpx.AsyncClient() as client:
        try:
            await asyncio.wait_for(client.get(url), timeout=1.0)
        except asyncio.TimeoutError:
            pass  # Expected
        
        # If pool exposes internals, assert in-flight count is 0
        # Example (adjust to actual pool API):
        # assert len(client._pool._connections_in_use) == 0
        
        # Fallback: absence of RuntimeError on exit verifies pool cleanup


@pytest.mark.asyncio
async def test_double_close_idempotency():
    """
    Verify calling aclose() multiple times does not raise an error.
    
    Risk 1 mitigation: Ensure aclose() is idempotent.
    """
    url = "https://httpbin.org/status/200"
    
    async with httpx.AsyncClient() as client:
        response = await client.get(url)
        
        # First close (explicit)
        await response.aclose()
        
        # Second close (should be no-op)
        await response.aclose()  # Should not raise


@pytest.mark.asyncio
async def test_cancel_during_context_manager():
    """
    Verify async with response context manager does not conflict with try/finally cleanup.
    
    Risk 4 mitigation: Ensure new cleanup integrates with __aexit__.
    """
    url = "https://httpbin.org/drip?duration=5"
    
    async with httpx.AsyncClient() as client:
        try:
            async with await client.get(url, stream=True) as response:
                # Cancel during body read inside context manager
                await asyncio.wait_for(response.aread(), timeout=1.0)
        except asyncio.TimeoutError:
            pass  # Expected
    
    # Implicit assertion: no double-close error or RuntimeError
```

**Test count:** 6 tests (minimum 4 required by AC-4, plus 2 for risk mitigations)

**Fixtures/helpers:**
- Use `httpbin.org` for slow endpoint (requires internet in CI).
- If `httpbin` unavailable, check `tests/conftest.py` for existing mock server fixtures.
- Fallback: create pytest fixture with `httpx.MockTransport` or `pytest-httpserver` (see Open Questions).

**Expected line count:** ~120-140 lines

---

#### 4. `tests/client/test_client_cancel.py` (conditional — only if sync client exists)

**Location:** `tests/client/test_client_cancel.py` in repo root  
**Purpose:** Mirror async tests for sync `Client` to verify AC-3 (sync/async parity)

**Conditional creation:** Only create if sync `Response.read()` and `Response.iter_bytes()` exist with separate logic (not just thin wrappers). See Open Questions Q3 for resolution.

**File structure (if created):**
```python
"""
Regression tests for response cleanup on sync cancel and timeout.

Verifies AC-3 (sync/async consistency).
"""

import threading
import pytest
import httpx


def test_sync_cancel_during_streaming_read():
    """
    Verify sync client cancel during streaming closes response cleanly.
    
    AC-3: Sync client should mirror async behavior.
    """
    url = "https://httpbin.org/drip?duration=5"
    
    # Use threading.Timer to simulate interrupt (sync equivalent of asyncio.wait_for)
    def timeout_handler():
        raise TimeoutError("Simulated timeout")
    
    with httpx.Client() as client:
        timer = threading.Timer(2.0, timeout_handler)
        timer.start()
        
        try:
            response = client.get(url)
            # Should be interrupted by timeout_handler
        except TimeoutError:
            pass  # Expected
        finally:
            timer.cancel()
    
    # Implicit assertion: no RuntimeError on exit


def test_sync_response_closed_after_cancel():
    """
    Verify sync response.is_closed is True after cancel.
    
    AC-3: Sync should mirror async behavior.
    """
    url = "https://httpbin.org/drip?duration=5"
    response = None
    
    def timeout_handler():
        raise TimeoutError("Simulated timeout")
    
    with httpx.Client() as client:
        timer = threading.Timer(1.0, timeout_handler)
        timer.start()
        
        try:
            response = client.get(url, stream=True)
            response.read()  # Should be interrupted
        except TimeoutError:
            pass  # Expected
        finally:
            timer.cancel()
    
    assert response is not None
    assert response.is_closed


def test_sync_double_close_idempotency():
    """
    Verify calling close() multiple times does not raise an error.
    
    AC-3: Sync should mirror async idempotency.
    """
    url = "https://httpbin.org/status/200"
    
    with httpx.Client() as client:
        response = client.get(url)
        response.close()  # First close
        response.close()  # Second close (should be no-op)


def test_sync_cancel_during_context_manager():
    """
    Verify sync with response context manager does not conflict with try/finally cleanup.
    
    AC-3: Sync should mirror async behavior.
    """
    url = "https://httpbin.org/drip?duration=5"
    
    def timeout_handler():
        raise TimeoutError("Simulated timeout")
    
    with httpx.Client() as client:
        timer = threading.Timer(1.0, timeout_handler)
        timer.start()
        
        try:
            with client.get(url, stream=True) as response:
                response.read()  # Should be interrupted
        except TimeoutError:
            pass  # Expected
        finally:
            timer.cancel()
```

**Test count:** 4 tests (mirroring async tests)

**Notes:**
- Sync timeout simulation is more complex than async (no `asyncio.wait_for` equivalent).
- `threading.Timer` + exception is one approach; may need adjustment based on httpx sync timeout mechanism.
- If sync client does not exist separately, **do not create this file**; document in Notes section that AC-3 is satisfied by shared response model.

**Expected line count (if created):** ~100-120 lines

---

### Audit-Only Files (read, no changes)

#### 5. `tests/client/test_async_client.py` (existing)

**Location:** `tests/client/test_async_client.py` in repo root  
**Purpose:** Verify no existing tests conflict with new cleanup logic

**Audit steps:**
1. Search for tests using `async with response` pattern.
2. Search for tests explicitly calling `response.aclose()`.
3. Verify new try/finally cleanup does not break assumptions in existing tests.

**Expected changes:** None (audit only; tests should continue to pass)

---

#### 6. `httpx/_client.py` or `httpx/_async_client.py` (audit only)

**Location:** `httpx/_client.py` or `httpx/_async_client.py` in httpx source  
**Purpose:** Verify client dispatcher does **not** own cleanup (per review feedback)

**Audit steps:**
1. Locate async request dispatcher (e.g., `AsyncClient._send_single_request()` or equivalent).
2. Confirm dispatcher does **not** catch `CancelledError`/`TimeoutError` and call `response.aclose()`.
3. Document that cleanup ownership is in Response methods, not dispatcher.

**Critical check:** If dispatcher **does** catch cancel/timeout and call `aclose()`, **remove that logic** as part of this fix (per review feedback: "The client dispatcher must not own this path").

**Expected changes:** None (audit only; cleanup is in Response methods per AD-1)

---

## Interfaces

### Response streaming interface (behavioral, no new API)

**Parties:** `Response.aread()` / `Response.aiter_bytes()` ↔ `Response.aclose()` ↔ connection pool

**Contract:**

**Input:** HTTP response stream (from `self._stream` or equivalent)

**Output:** Response body bytes (from `aread()` or `aiter_bytes()`)

**Behavioral guarantee (post-fix):**
- **Normal completion:** Body read completes → `finally` block calls `aclose()` → stream closed, connection returned to pool → method returns body
- **Cancel/timeout:** `CancelledError` or `TimeoutError` raised mid-read → `finally` block calls `aclose()` → stream closed, connection returned to pool → exception propagates
- **Double-close safety:** `aclose()` is idempotent; calling multiple times (e.g., from `finally` + `__aexit__`) does not raise error

**No new public methods or attributes.**

---

### Pool release shielding interface (internal)

**Parties:** `Response.aclose()` ↔ connection pool

**Contract:**

**Input:** `Response.aclose()` called (may be during cancel)

**Output:** Connection returned to pool atomically

**Behavioral guarantee (post-fix):**
- Pool release operation (`await self._pool.release(self._connection)`) wrapped in `asyncio.shield()`
- If second `CancelledError` fires during `aclose()` execution, pool release completes before propagating exception
- Connection is **never** left in in-flight state after `aclose()` returns (even if `aclose()` was cancelled)

**No new API; internal implementation change in `aclose()`.**

---

## Algorithm Notes

### Async response cleanup flow

**`Response.aread()` execution:**

1. Enter `aread()` method
2. `try:` → start body read loop
   - Call `self.aiter_bytes()` (or consume `self._stream` directly)
   - Accumulate chunks into `body` variable
3. If cancel/timeout fires → `CancelledError` or `TimeoutError` raised → jump to `finally`
4. `finally:` → call `await self.aclose()`
   - `aclose()` checks `if self.is_closed: return` (idempotency guard)
   - Close underlying stream: `await self._stream.aclose()`
   - Shield pool release: `await asyncio.shield(self._pool.release(self._connection))`
5. Exception propagates to caller (or method returns `body` if no exception)

**`Response.aiter_bytes()` execution:**

1. Enter `aiter_bytes()` generator
2. `try:` → start iteration loop
   - `async for chunk in self._stream:` → yield chunks
3. If cancel/timeout fires → `CancelledError` or `TimeoutError` raised → generator closes → jump to `finally`
4. `finally:` → call `await self.aclose()` (same cleanup as `aread()`)
5. Exception propagates to caller (or generator exhausts if no exception)

**`Response.aclose()` execution (with shielding):**

1. Check idempotency: `if self.is_closed: return`
2. Mark closed: `self._is_closed = True`
3. Close stream: `await self._stream.aclose()` (may be no-op if already closed)
4. Shield pool release:
   ```python
   if self._connection is not None:
       await asyncio.shield(self._pool.release(self._connection))
   ```
   - If `CancelledError` fires during `release()` call, `asyncio.shield()` ensures `release()` completes before propagating exception
   - Connection is atomically returned to pool

**Critical invariant:** After `aclose()` returns (normally or via exception), connection is **never** left in in-flight state.

---

### Sync response cleanup flow (parallel logic)

**`Response.read()` execution (sync):**

1. `try:` → body read loop
2. If interrupt (e.g., `KeyboardInterrupt`) → jump to `finally`
3. `finally:` → call `self.close()`
4. Exception propagates (or return `body`)

**`Response.close()` execution (sync):**

1. Idempotency check
2. Close stream
3. Return connection to pool (sync; non-cancellable, but wrap in try/finally for safety)

**Sync equivalent of shielding:** Sync operations are not cancellable in the same way as async, but pool release should be wrapped in `try/finally` to ensure it completes even if cleanup itself raises.

---

## Tests

### Test 1: `test_cancel_during_streaming_read`

**File:** `tests/client/test_async_client_cancel.py`

**Purpose:** Verify cancel via `asyncio.wait_for` timeout closes response and returns connection to pool

**Setup:**
- Create `AsyncClient`
- GET slow endpoint (`https://httpbin.org/drip?duration=5`)

**Action:**
- Wrap in `asyncio.wait_for(..., timeout=2.0)`
- Timeout fires mid-stream (2s < 5s duration)

**Assert:**
- `asyncio.TimeoutError` raised
- `async with client` context exit completes without `RuntimeError`

**Verifies:** AC-1, AC-2

---

### Test 2: `test_client_timeout_during_streaming_read`

**File:** `tests/client/test_async_client_cancel.py`

**Purpose:** Verify client timeout (not `asyncio.wait_for`) closes response cleanly

**Setup:**
- Create `AsyncClient(timeout=2.0)`
- GET slow endpoint

**Action:**
- Let client timeout fire (not external `wait_for`)

**Assert:**
- `httpx.TimeoutException` raised
- Client context exit completes without `RuntimeError`

**Verifies:** AC-1, AC-2

---

### Test 3: `test_response_closed_after_cancel`

**File:** `tests/client/test_async_client_cancel.py`

**Purpose:** Verify `response.is_closed` is `True` after cancel

**Setup:**
- Create `AsyncClient`
- GET slow endpoint with `stream=True`

**Action:**
- Start reading body: `await response.aread()`
- Wrap in `asyncio.wait_for(..., timeout=1.0)`
- Timeout fires

**Assert:**
- `response is not None`
- `response.is_closed == True`

**Verifies:** AC-1 (response closure)

---

### Test 4: `test_connection_returned_to_pool_after_cancel`

**File:** `tests/client/test_async_client_cancel.py`

**Purpose:** Verify connection pool state is clean after cancel

**Setup:**
- Create `AsyncClient`
- GET slow endpoint

**Action:**
- Cancel via `asyncio.wait_for`
- Inspect pool state (if exposed)

**Assert:**
- In-flight connection count is 0 (or connections available for reuse)
- **Fallback:** Absence of `RuntimeError` in tests 1-2 verifies this indirectly

**Verifies:** AC-1 (pool return)

**Note:** May require pool internals access (`client._pool._connections_in_use` or similar). If not exposed, test is satisfied by absence of `RuntimeError` in other tests.

---

### Test 5: `test_double_close_idempotency`

**File:** `tests/client/test_async_client_cancel.py`

**Purpose:** Verify calling `aclose()` multiple times does not raise error

**Setup:**
- Create `AsyncClient`
- GET any endpoint

**Action:**
- Call `await response.aclose()` twice

**Assert:**
- Second call does not raise exception

**Verifies:** Risk 1 mitigation (idempotency)

---

### Test 6: `test_cancel_during_context_manager`

**File:** `tests/client/test_async_client_cancel.py`

**Purpose:** Verify `async with response` context manager does not conflict with try/finally cleanup

**Setup:**
- Create `AsyncClient`
- GET slow endpoint with `stream=True`

**Action:**
- Use `async with response:` context manager
- Cancel during body read inside context

**Assert:**
- No double-close error
- No `RuntimeError` on client exit

**Verifies:** Risk 4 mitigation (integration with `__aexit__`)

---

### Sync tests (if applicable)

**File:** `tests/client/test_client_cancel.py`

**Tests 7-10:** Mirror tests 1-4 for sync `Client`:
- `test_sync_cancel_during_streaming_read`
- `test_sync_response_closed_after_cancel`
- `test_sync_double_close_idempotency`
- `test_sync_cancel_during_context_manager`

**Verifies:** AC-3 (sync/async consistency)

**Note:** Only create if sync `Client` exists with separate streaming logic. See Open Questions Q3.

---

### Test Summary

**Total test count:**
- **Async:** 6 tests (4 required by AC-4, plus 2 for risk mitigations)
- **Sync:** 4 tests (if sync client exists separately)
- **Total:** 6-10 tests

**Coverage:**
- **AC-1:** Tests 1, 2, 3, 4
- **AC-2:** Tests 1, 2
- **AC-3:** Tests 7-10 (sync mirror tests, if applicable)
- **AC-4:** All tests above

---

## Open Questions & Resolutions

### Q1: Does the current `Response.aclose()` implementation shield pool release from cancellation?

**Status:** Must be resolved during implementation

**Resolution path:**
1. Locate `httpx/_models.py` → `Response.aclose()` method
2. Trace pool release call (e.g., `await self._pool.release(self._connection)`)
3. Check if wrapped in `asyncio.shield()` or equivalent
4. **If not shielded:** Add `asyncio.shield()` as part of this fix (see File Plan Part C)
5. **If already shielded:** Document and proceed

**Impact:** Medium (Risk 2 mitigation depends on this)

**Blocking:** No (proceed with fix assumption that shielding is needed)

---

### Q2: Are there existing try/finally blocks in `Response.aread()` or `Response.aiter_bytes()`?

**Status:** Must be resolved during implementation

**Resolution path:**
1. Inspect `httpx/_models.py` → `Response.aread()` and `Response.aiter_bytes()`
2. Search for existing `try/finally` blocks around body read operations
3. **If existing cleanup found:**
   - Verify it calls `aclose()` in `finally` block
   - If yes, no change needed (cleanup already correct)
   - If no, integrate new `aclose()` call into existing `finally`
4. **If no existing try/finally:** Add new try/finally wrapper (see File Plan Parts A and B)

**Impact:** Low (integration approach depends on this, but fix is additive)

**Blocking:** No (default assumption: no existing try/finally, add new wrapper)

---

### Q3: Does sync `Client` exist as a separate implementation?

**Status:** Must be resolved during implementation

**Resolution path:**
1. Open `httpx/_models.py` and search for `def read(self)` (non-async)
2. Check if sync `Response.read()` and `Response.iter_bytes()` have separate body-read logic
3. **If separate logic exists:** Apply parallel fix (try/finally + `self.close()`), create `test_client_cancel.py`
4. **If sync methods do not exist or are thin wrappers:** Document in Notes, skip sync tests, mark AC-3 as satisfied by shared response model

**Impact:** High (AC-3 depends on this)

**Blocking:** No (async fix proceeds regardless; sync fix is conditional)

**Expected outcome:** Sync methods likely exist but may share core logic with async. Verify during audit.

---

### Q4: What test fixture is used for slow/drip endpoints?

**Status:** Must be resolved during test implementation

**Resolution path:**
1. Check `tests/conftest.py` for existing slow-response fixtures
2. Check `tests/common.py` for httpbin or mock server helpers
3. **If fixture exists:** Use it in new tests
4. **If no fixture:** Use `https://httpbin.org/drip?duration=5` (requires internet in CI)
5. **Fallback:** Create pytest fixture with `httpx.MockTransport` or `pytest-httpserver`

**Impact:** Low (test setup depends on this, but httpbin is acceptable default)

**Blocking:** No (use httpbin initially; optimize fixture later if needed)

---

### Q5: Does the client dispatcher currently own any cancel handling that conflicts with response-owned cleanup?

**Status:** Must be resolved during audit

**Resolution path:**
1. Audit `httpx/_client.py` or `httpx/_async_client.py` → locate request dispatcher (e.g., `AsyncClient._send_single_request()`)
2. Search for `try/except` blocks catching `CancelledError` or `Time
