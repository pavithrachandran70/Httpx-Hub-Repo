# Merged plan — PER-81

Execution order (1 ticket(s)):

1. `PER-82`

---

## Ticket `PER-82`

# Tech Stub — PER-82

**Ticket ID:** PER-82  
**Story:** Implement response cleanup on async cancel/timeout with sync parity and tests  
**Type:** Bug  
**Repo:** Https-test  
**Altitude:** Story  
**Date:** 2025-01-23

---

## Stack & Constraints

**Repository:** Https-test  
**Base branch:** `master`  
**Remote:** https://github.com/pavithrachandran70/httpx-demo  
**Language/Framework:** Python (inferred from `pyproject.toml`, `tests/` structure)  
**Test runner:** pytest (inferred from existing test structure in `tests/client/test_async_client.py`)  
**Lint:** Not specified

**Key constraints:**
- httpx source code is **not present** in the provided repo tree (`/home/pavc/Documents/Https-Demo` contains only `tests/` directory)
- Implementer must locate httpx source files in site-packages or clone httpx repository separately
- Primary changes target `httpx/_models.py` (Response.aread, Response.aiter_bytes, Response.aclose), not client dispatcher per review feedback
- Existing test suite structure: `tests/client/test_async_client.py` for async client tests

---

## Schema & API Contracts

### 1. Response streaming cleanup contract (behavioral, no new API)

**Parties:** `Response.aread()` / `Response.aiter_bytes()` ↔ `Response.aclose()`

**Existing behavior (broken):**
- `Response.aread()` and `Response.aiter_bytes()` read response body chunks
- When `CancelledError` or `TimeoutError` fires during read, methods raise exception without cleanup
- Response stream is **not** closed
- Connection remains marked as in-flight
- `async with AsyncClient` exit raises `RuntimeError: The connection pool was closed while 1 HTTP requests/responses were still in-flight.`

**Required behavior (fix):**
- `Response.aread()` and `Response.aiter_bytes()` wrap body read in `try/finally`
- `finally` block always calls `await self.aclose()`, even on cancel/timeout
- `response.aclose()` closes stream and returns connection to pool atomically with shielded pool release
- `async with AsyncClient` exit completes cleanly, no `RuntimeError`

**Method signatures (existing, no change):**
```python
class Response:
    async def aread(self) -> bytes:
        """Read entire response body. Post-fix: wraps in try/finally, calls aclose() on exit."""
        ...
    
    async def aiter_bytes(self, chunk_size: int = None) -> AsyncIterator[bytes]:
        """Iterate response body chunks. Post-fix: wraps iteration in try/finally, calls aclose() on exit."""
        ...
    
    async def aclose(self) -> None:
        """Close response stream and return connection to pool. Idempotent. Post-fix: shields pool release."""
        ...
```

### 2. Pool release shielding contract

**Parties:** `Response.aclose()` ↔ connection pool

**Contract:**
- `Response.aclose()` must shield the pool release operation with `asyncio.shield()`
- Pool release completes atomically even if `aclose()` is cancelled during execution
- Prevents connection leak when second cancel fires during cleanup

**No new API; internal implementation change in `aclose()`**

### 3. Sync/async parity contract

**Parties:** Sync `Response.read()` / `Response.iter_bytes()` ↔ Async `Response.aread()` / `Response.aiter_bytes()`

**Contract:**
- Both sync and async response streaming methods follow identical cleanup pattern
- Sync: wrap body read in `try/finally`, call `self.close()` in finally block
- Async: wrap body read in `try/finally`, call `await self.aclose()` in finally block
- Both suppress in-flight connection error on client context exit after cancel/timeout

**No new API; behavioral alignment only.**

---

## File Touch List

### Implementation files (httpx source — not in repo tree)

**Note:** The httpx library source is **not** present in the provided repo tree (`/home/pavc/Documents/Https-Demo`). The implementer must locate httpx source files via:
- `python -c "import httpx; print(httpx.__file__)"` to find installed location, or
- Clone httpx repository separately for modification

Paths below are inferred from standard httpx structure and must be verified at implementation time.

#### 1. **`httpx/_models.py`** (Response class — primary fix location)

**Location inference:** httpx source root → `httpx/_models.py`  
**Purpose:** Add try/finally cleanup to Response.aread and Response.aiter_bytes; shield pool release in Response.aclose  
**Changes:**

**Part A: Response.aread() cleanup**
- Locate `async def aread(self) -> bytes:` method
- Wrap the body read operation in `try/finally` block
- In `finally` block, call `await self.aclose()`

**Pseudocode (conceptual):**
```python
async def aread(self) -> bytes:
    try:
        # existing body read logic (e.g., consume self._stream, accumulate chunks)
        body = b""
        async for chunk in self.aiter_bytes():
            body += chunk
        return body
    finally:
        await self.aclose()
```

**Part B: Response.aiter_bytes() cleanup**
- Locate `async def aiter_bytes(self, chunk_size: int = None) -> AsyncIterator[bytes]:` method
- Wrap the iteration logic in `try/finally` block
- In `finally` block, call `await self.aclose()`

**Pseudocode (conceptual):**
```python
async def aiter_bytes(self, chunk_size: int = None) -> AsyncIterator[bytes]:
    try:
        # existing iteration logic (yield chunks from self._stream)
        async for chunk in self._stream:
            yield chunk
    finally:
        await self.aclose()
```

**Part C: Response.aclose() shielding**
- Locate `async def aclose(self) -> None:` method
- Verify idempotency guard exists (e.g., `if self._closed: return`)
- Wrap pool release call in `asyncio.shield()` to prevent cancellation during cleanup

**Pseudocode (conceptual):**
```python
async def aclose(self) -> None:
    if self._closed:
        return  # idempotent guard
    self._closed = True
    
    # Close underlying stream
    if self._stream is not None:
        await self._stream.aclose()
    
    # Shield pool release from cancellation
    if self._connection is not None:
        await asyncio.shield(self._pool.release(self._connection))
```

**Expected line delta:** +30-40 lines (try/finally in aread and aiter_bytes, shield in aclose, idempotency guard if missing)

#### 2. **`httpx/_models.py`** (Response class — sync equivalents)

**Location inference:** Same file, sync methods in `Response` class  
**Purpose:** Apply parallel cleanup pattern to sync `read()` and `iter_bytes()` methods  
**Changes:**

**Part A: Response.read() cleanup (sync)**
```python
def read(self) -> bytes:
    try:
        # existing body read logic
        body = b""
        for chunk in self.iter_bytes():
            body += chunk
        return body
    finally:
        self.close()
```

**Part B: Response.iter_bytes() cleanup (sync)**
```python
def iter_bytes(self, chunk_size: int = None) -> Iterator[bytes]:
    try:
        # existing iteration logic
        for chunk in self._stream:
            yield chunk
    finally:
        self.close()
```

**Part C: Response.close() verification (sync)**
- Verify `close()` is idempotent (same guard as `aclose()`)
- Verify pool release is wrapped in try/finally (sync equivalent of shielding)

**Expected line delta:** +20-30 lines (try/finally in read and iter_bytes, verification of close)

#### 3. **`httpx/_async_pool.py`** (connection pool — audit only)

**Location inference:** httpx source root → `httpx/_async_pool.py` or `httpx/_pool.py`  
**Purpose:** Verify pool's `release()` method correctly decrements in-flight count when called  
**Audit steps:**
- Locate pool's `async def release(self, connection)` method
- Confirm it decrements in-flight counter (e.g., `self._in_flight -= 1`)
- Confirm pool does **not** detect cancel/timeout itself (only receives release signals)

**Expected changes:** None (audit only; document findings in Notes section)

---

### Test files (created in repo tree)

#### 4. **`tests/client/test_async_client_cancel.py`** (new file)

**Purpose:** Regression tests for async cancel and timeout scenarios  
**Tests to implement:**

**Test 1: `test_cancel_during_streaming_read`**
- **Setup:** `AsyncClient`, GET to slow endpoint (`https://httpbin.org/drip?duration=5`)
- **Action:** Wrap in `asyncio.wait_for(client.get(url), timeout=2)` (timeout < duration)
- **Assert:** `asyncio.TimeoutError` raised; `async with client` exits cleanly (no `RuntimeError`)
- **Verifies:** AC-1, AC-2

**Test 2: `test_client_timeout_during_streaming_read`**
- **Setup:** `AsyncClient(timeout=2.0)`, GET to slow endpoint
- **Action:** Let client timeout fire (not `asyncio.wait_for`)
- **Assert:** `httpx.TimeoutException` raised; client exits cleanly
- **Verifies:** AC-1, AC-2

**Test 3: `test_response_closed_after_cancel`**
- **Setup:** Start streaming read, cancel via `asyncio.wait_for`
- **Action:** Catch exception, inspect `response.is_closed` or equivalent
- **Assert:** `response.is_closed == True`
- **Verifies:** AC-1 (response closure)

**Test 4: `test_connection_returned_to_pool_after_cancel`**
- **Setup:** Start streaming, cancel mid-stream
- **Action:** Inspect pool state (if `client._pool` exposes internals)
- **Assert:** In-flight count is 0 (or connection available)
- **Verifies:** AC-1 (pool return)

**Expected line count:** ~100-120 lines (4 tests, setup/teardown, fixtures)

#### 5. **`tests/client/test_client_cancel.py`** (new file, if sync client exists)

**Purpose:** Mirror async tests for sync `Client` to verify AC-3 (sync/async parity)  
**Conditional creation:** Only if sync `Client` has separate streaming read logic  
**Tests:** Same as async tests, adapted for sync (replace `async with` → `with`, `await` → blocking calls)

**Expected line count:** ~100-120 lines (or 0 if sync client does not exist separately)

---

### Audit-only files (read, no expected changes)

#### 6. **`tests/client/test_async_client.py`** (existing)

**Purpose:** Verify no existing tests conflict with new cleanup logic  
**Audit steps:**
- Search for tests using `async with response` or calling `response.aclose()` explicitly
- Verify new cleanup does not break assumptions in existing tests

**Expected changes:** None (audit only)

#### 7. **`httpx/_client.py`** or `httpx/_async_client.py`** (audit only)

**Purpose:** Verify client dispatcher does **not** own cleanup (per review feedback)  
**Audit steps:**
- Locate async request dispatcher (e.g., `AsyncClient._send_single_request()`)
- Confirm dispatcher does **not** catch `CancelledError`/`TimeoutError` and call `aclose()`
- Document that cleanup ownership is in Response methods, not dispatcher

**Expected changes:** None (cleanup is in Response methods per AD-1)

---

## Risks

### Risk 1: Cancel raised after response.aclose() already called
**Likelihood:** Low  
**Impact:** Low (double-close is safe if `aclose()` is idempotent)  
**Mitigation:**
- Verify `response.aclose()` has idempotency guard in `httpx/_models.py`
- Add test case: call `aclose()` twice, assert no exception
- **Detection:** Code audit of `Response.aclose()` implementation
- **Fallback:** Add `if self._closed: return` guard if missing

### Risk 2: Second cancel fires during aclose() execution
**Likelihood:** Low  
**Impact:** Medium (connection might not return to pool if pool release is interrupted)  
**Mitigation:**
- Shield pool release inside `aclose()` with `asyncio.shield()`
- Add test: cancel during `aclose()` execution (requires timing or mock), verify pool state
- **Detection:** Trace `aclose()` → pool release code path; check for shield
- **Fallback:** Wrap pool release in `asyncio.shield()` (part of primary fix)

### Risk 3: Sync client diverges from async client in cleanup logic
**Likelihood:** Medium (if fix only touches async methods)  
**Impact:** High (AC-3 failure)  
**Mitigation:**
- Apply parallel fix to sync `Response.read()` and `Response.iter_bytes()` in `httpx/_models.py`
- Add sync tests in `test_client_cancel.py` mirroring async tests
- **Detection:** Verify sync methods exist and have separate logic
- **Fallback:** None; sync fix is mandatory per AC-3

### Risk 4: Existing code assumes response auto-closes on context exit only
**Likelihood:** Medium  
**Impact:** Medium (potential conflict with `__aexit__` cleanup)  
**Mitigation:**
- Audit `Response.__aexit__` to verify it calls `aclose()`
- Ensure try/finally cleanup in streaming methods does not conflict (both call same `aclose()`)
- Add test: use `async with response` after cancel, verify no double-close error
- **Detection:** Code audit + test
- **Fallback:** Ensure `aclose()` idempotency handles overlap

### Risk 5: httpx source not available in repo
**Likelihood:** High (confirmed by repo tree)  
**Impact:** Blocking  
**Mitigation:**
- Implementer must locate httpx via `python -c "import httpx; print(httpx.__file__)"` or clone httpx repository
- Document actual httpx source path in implementation notes
- **Detection:** Immediate (repo tree shows no httpx source)
- **Fallback:** Install httpx in editable mode or modify in site-packages

---

## Open Questions

### Q1: Does the current Response.aclose() implementation shield pool release from cancellation?
**Context:** If `aclose()` can be cancelled mid-execution, connection might not return to pool  
**Resolution path:**
1. Locate `httpx/_models.py` → `Response.aclose()` implementation
2. Trace pool release call (e.g., `await self._pool.release(self._connection)`)
3. Check if wrapped in `asyncio.shield()` or try/finally
4. If not shielded, add `asyncio.shield()` as part of this fix (Risk 2 mitigation)
5. Document findings in implementation notes

**Blocking:** No (proceed with fix, add shielding if audit reveals it's missing)

### Q2: Are Response.aread() and Response.aiter_bytes() currently wrapped in try/finally?
**Context:** Fix might conflict with existing cleanup logic  
**Resolution path:**
1. Inspect `httpx/_models.py` → `Response.aread()` and `Response.aiter_bytes()`
2. Search for existing `try/finally` blocks around body read operations
3. If existing cleanup found, verify it calls `aclose()` in finally
4. Integrate new cleanup without duplication (use existing finally block if present)

**Blocking:** No (audit informs implementation approach, does not block starting)

### Q3: Does sync Client exist as separate implementation with its own streaming methods?
**Context:** AC-3 requires sync/async parity; need to confirm sync methods exist  
**Resolution path:**
1. Open `httpx/_models.py` and search for `def read(self)` and `def iter_bytes(self)` (non-async)
2. If found, verify they have separate body read logic (not just wrappers)
3. If separate logic exists, apply parallel fix (try/finally + `self.close()`)
4. If sync methods do not exist or are deprecated, document as N/A and skip sync tests

**Blocking:** No (async fix proceeds regardless; sync fix is conditional)

### Q4: What test fixture is used for slow/drip endpoints in existing tests?
**Context:** Tests need slow-response endpoint; prefer existing fixture over external dependency  
**Resolution path:**
1. Check `tests/conftest.py` for slow-response fixtures
2. Check `tests/common.py` for httpbin or mock server helpers
3. If no fixture exists, use `https://httpbin.org/drip?duration=5` (requires internet in CI)
4. Fallback: create pytest fixture with `httpx.MockTransport` or `pytest-httpserver`

**Blocking:** No (tests can use httpbin.org initially; optimize fixture later)

---

## Test Coverage (per AC-4)

### Required regression tests

**File:** `tests/client/test_async_client_cancel.py`

**Test 1: `test_cancel_during_streaming_read`**
```python
import asyncio
import httpx
import pytest

@pytest.mark.asyncio
async def test_cancel_during_streaming_read():
    """Verify cancel during streaming closes response and returns connection to pool."""
    url = "https://httpbin.org/drip?duration=5"
    async with httpx.AsyncClient() as client:
        with pytest.raises(asyncio.TimeoutError):
            await asyncio.wait_for(client.get(url), timeout=2)
    # Assert: no RuntimeError raised during client context exit (implicit)
```

**Test 2: `test_client_timeout_during_streaming_read`**
```python
@pytest.mark.asyncio
async def test_client_timeout_during_streaming_read():
    """Verify client timeout during streaming closes response cleanly."""
    url = "https://httpbin.org/drip?duration=5"
    async with httpx.AsyncClient(timeout=2.0) as client:
        with pytest.raises(httpx.TimeoutException):
            await client.get(url)
    # No RuntimeError on exit
```

**Test 3: `test_response_closed_after_cancel`**
```python
@pytest.mark.asyncio
async def test_response_closed_after_cancel():
    """Verify response.is_closed is True after cancel."""
    url = "https://httpbin.org/drip?duration=5"
    response = None
    async with httpx.AsyncClient() as client:
        try:
            response = await asyncio.wait_for(
                client.get(url, stream=True), timeout=1
            )
            # Start reading to trigger cleanup
            await asyncio.wait_for(response.aread(), timeout=0.5)
        except asyncio.TimeoutError:
            pass
    
    # Response should be closed after cancel
    assert response is not None
    assert response.is_closed  # or equivalent attribute
```

**Test 4: `test_connection_returned_to_pool_after_cancel`**
```python
@pytest.mark.asyncio
async def test_connection_returned_to_pool_after_cancel():
    """Verify connection pool state is clean after cancel."""
    url = "https://httpbin.org/drip?duration=5"
    async with httpx.AsyncClient() as client:
        try:
            await asyncio.wait_for(client.get(url), timeout=1)
        except asyncio.TimeoutError:
            pass
        # If pool exposes internals, assert in-flight count is 0
        # Otherwise, absence of RuntimeError on exit verifies this
```

### Sync tests (conditional)

**File:** `tests/client/test_client_cancel.py` (only if sync Client exists separately)

**Tests 1-4:** Mirror async tests above, adapted for sync:
- Replace `async with` → `with`
- Replace `await` → blocking calls
- Replace `asyncio.TimeoutError` → sync timeout exception
- Use threading or signal-based interrupt mechanism

### Coverage verification
- **AC-1 (cancel/timeout closes response and returns connection):** Tests 1, 2, 3, 4
- **AC-2 (no RuntimeError on client exit):** Tests 1, 2
- **AC-3 (sync/async consistency):** Sync mirror tests (if applicable)
- **AC-4 (regression tests exist):** All tests above

**Total test count:** **4 async tests** (required) + **4 sync tests** (if sync Client exists separately) = **4-8 tests**

---

## Implementation Notes

### Primary fix location

**File:** `httpx/_models.py` (Response class)  
**Core changes:**
1. Wrap `Response.aread()` body read in `try/finally`, call `await self.aclose()` in finally
2. Wrap `Response.aiter_bytes()` iteration in `try/finally`, call `await self.aclose()` in finally
3. Add `asyncio.shield()` around pool release in `Response.aclose()`
4. Apply parallel changes to sync `Response.read()` and `Response.iter_bytes()` (if exist)

### Not the client dispatcher

**Per review feedback:** Do **not** catch `CancelledError`/`TimeoutError` in `AsyncClient._send_single_request()` or equivalent dispatcher. Cleanup ownership is in `Response.aread()` and `Response.aiter_bytes()`, not the client dispatcher.

### Idempotency and shielding

- **Idempotency:** `Response.aclose()` must check `if self._closed: return` to prevent double-close errors
- **Shielding:** Pool release in `aclose()` must be wrapped in `asyncio.shield()` to prevent connection leak if second cancel fires during cleanup

### httpx source location

- Run `python -c "import httpx; print(httpx.__file__)"` to find installed httpx location
- Expected output: `/path/to/site-packages/httpx/__init__.py` → source is in same directory
- Modify `httpx/_models.py` in that location, or clone httpx repository for development

---

**End of Tech Stub.**

---

# Story summaries

- `01-Https-test`: LLD — Story 01-Https-test: Implement response cleanup on async cancel/timeout with sync parity and tests

---

_Single-ticket pack: order is trivial (no multi-split)._
