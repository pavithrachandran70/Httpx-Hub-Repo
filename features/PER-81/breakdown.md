# Breakdown — Close response stream when async request is cancelled mid-read

**Ticket:** PER-81  
**Type:** Bug  
**Epic:** (standalone)  
**Date:** 2025-01-23

---

## Overview

This breakdown converts the HLD architectural decisions and requirements into a single implementable story. The story addresses the core issue: when an `AsyncClient` request is cancelled or times out during response streaming, the response stream is not closed, leaving the connection marked as in-flight and causing a `RuntimeError` when the client exits.

**Key architectural change (per review feedback):** Response cleanup ownership is in `Response.aread()` and `Response.aiter_bytes()` with try/finally blocks, **not** in the AsyncClient dispatcher. The response methods wrap body reads in try/finally and always call `aclose()` on exit (including cancel and timeout). `Response.aclose()` shields the pool release operation with `asyncio.shield()` to ensure the connection is returned even if a second cancel fires during cleanup.

---

## Explicit Ordered Execution List

### Story 1: Implement response cleanup on async cancel/timeout with sync parity and tests

**Repo:** `Https-test`

**Description:**

When an `AsyncClient` request is cancelled (via `asyncio.wait_for` timeout or explicit cancellation) or times out (via client timeout) while the response body is streaming, the response stream must be closed and the connection returned to the pool before the exception propagates. This story implements cleanup ownership at the Response method level (not the client dispatcher), where `Response.aread()` and `Response.aiter_bytes()` wrap body reads in try/finally blocks and always call `aclose()` on exit. The `Response.aclose()` method shields pool release with `asyncio.shield()` to prevent connection leaking if a second cancel fires during cleanup. The sync `Client` must exhibit identical cleanup behavior. This story also adds comprehensive regression tests for all cancel/timeout scenarios.

**Covers:**
- **AC-1:** Response closure on cancel
- **AC-2:** Response closure on timeout  
- **AC-3:** Sync/async consistency
- **AC-4:** Regression test coverage

**Prerequisites:** None (standalone bug fix)

**Depends on (after):** None

**Architectural Decisions Applied:**
- **AD-1 (revised):** Response cleanup ownership — `Response.aread()` and `Response.aiter_bytes()` wrap body read in try/finally, call `aclose()` in finally block
- **AD-2:** Response stream closure mechanism — `response.aclose()` is the single entry point for cleanup, idempotent
- **AD-3:** Sync/async consistency — sync `Response.read()` and `Response.iter_bytes()` mirror async cleanup logic
- **AD-4 (revised):** Pool release shielding — `Response.aclose()` shields the connection pool release operation with `asyncio.shield()`

**Implementation approach:**

1. **Async cleanup in Response.aread() (primary fix):**
   - Locate `Response.aread()` in `httpx/_models.py` (or equivalent Response class file in httpx source).
   - Wrap the body read operation (likely a loop consuming `self._stream` or calling `self.aiter_bytes()`) in a `try/finally` block.
   - In the `finally` block, call `await self.aclose()` unconditionally.
   - Ensure the fix does not conflict with existing cleanup (audit for existing `try/finally` or `__aexit__` behavior).

   **Pseudocode:**
   ```python
   async def aread(self) -> bytes:
       try:
           # existing body read logic (e.g., accumulate chunks from aiter_bytes())
           body = b""
           async for chunk in self.aiter_bytes():
               body += chunk
           return body
       finally:
           await self.aclose()
   ```

2. **Async cleanup in Response.aiter_bytes() (primary fix):**
   - Locate `Response.aiter_bytes()` in `httpx/_models.py`.
   - Wrap the iteration logic (likely yielding chunks from `self._stream`) in a `try/finally` block.
   - In the `finally` block, call `await self.aclose()` unconditionally.

   **Pseudocode:**
   ```python
   async def aiter_bytes(self, chunk_size: int = None) -> AsyncIterator[bytes]:
       try:
           # existing iteration logic (yield chunks from self._stream)
           async for chunk in self._stream:
               yield chunk
       finally:
           await self.aclose()
   ```

3. **Shield pool release in Response.aclose():**
   - Locate `Response.aclose()` in `httpx/_models.py`.
   - Verify idempotency guard exists (e.g., `if self._closed: return`); add if missing.
   - Locate the pool release call (likely `await self._pool.release(self._connection)` or similar).
   - Wrap the pool release operation in `asyncio.shield()` to ensure it completes even if `aclose()` is cancelled mid-execution.

   **Pseudocode:**
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

4. **Sync cleanup (parity fix):**
   - Locate sync `Response.read()` and `Response.iter_bytes()` in `httpx/_models.py` (same file, non-async methods).
   - Apply parallel logic: wrap body read in `try/finally`, call `self.close()` in finally block.
   - Verify `Response.close()` (sync) is idempotent and wraps pool release in try/finally (sync equivalent of shielding).

   **Pseudocode (sync read):**
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

   **Pseudocode (sync iter_bytes):**
   ```python
   def iter_bytes(self, chunk_size: int = None) -> Iterator[bytes]:
       try:
           # existing iteration logic
           for chunk in self._stream:
               yield chunk
       finally:
           self.close()
   ```

5. **Connection pool audit (verify, no change expected):**
   - Open `httpx/_async_pool.py` (or equivalent pool module).
   - Verify the pool's connection release logic correctly decrements the in-flight count when `response.aclose()` returns a connection.
   - Confirm the pool does **not** attempt to detect cancel/timeout itself (per AD-4); it should only receive release signals from response objects.
   - Document findings; no code change expected unless a discrepancy is found.

6. **Regression tests (async):**
   - Create `tests/client/test_async_client_cancel.py`.
   - Implement the following tests:
     - **`test_cancel_during_streaming_read`**: Use `asyncio.wait_for(client.get(url), timeout=3)` with a slow endpoint (e.g., `https://httpbin.org/drip?duration=5`). Assert `asyncio.TimeoutError` is raised and `async with client` exits without `RuntimeError`.
     - **`test_client_timeout_during_streaming_read`**: Configure `AsyncClient(timeout=3)`, GET a slow endpoint, let the client timeout fire (not `asyncio.wait_for`). Assert `httpx.TimeoutException` (or equivalent) is raised and client exits cleanly.
     - **`test_response_closed_after_cancel`**: Cancel during streaming, catch exception, assert `response.is_closed` (or equivalent property) is `True`.
     - **`test_connection_returned_to_pool_after_cancel`**: If pool exposes in-flight count (e.g., `client._pool._connections` or similar), verify count returns to 0 after cancel. If not exposed, this test is satisfied by the absence of `RuntimeError` in test 1.
   - Use a test fixture or mock server that provides a slow/drip endpoint if `httpbin.org` is not available in CI (check existing test setup in `tests/conftest.py` or `tests/common.py`).

7. **Regression tests (sync, if applicable):**
   - If the sync `Client` exists as a separate implementation (check `httpx/_client.py` for sync methods), create `tests/client/test_client_cancel.py`.
   - Mirror the async tests above, adapted for sync:
     - Use `threading.Timer` or signal-based timeout to simulate interrupt.
     - Replace `asyncio.TimeoutError` with sync equivalents (e.g., `TimeoutError` or `KeyboardInterrupt` depending on httpx sync timeout mechanism).
     - Replace `async with` with `with` and `await` with blocking calls.
   - If sync client does not exist or is deprecated, document this in Notes and skip sync tests (but still verify AC-3 is not violated by checking if sync code paths exist at all).

**Allowed paths:**

Primary changes (must exist or be located):
- `httpx/_models.py` (Response class) — add try/finally in `aread()` and `aiter_bytes()`, shield pool release in `aclose()`, parallel changes to sync `read()` and `iter_bytes()`

Test files (created):
- `tests/client/test_async_client_cancel.py` (new file, 4 tests minimum)
- `tests/client/test_client_cancel.py` (new file if sync client exists, 4 tests mirroring async)

Audit-only (read, no expected changes):
- `httpx/_async_pool.py` (verify pool release logic)
- `tests/client/test_async_client.py` (check for conflicting existing tests)
- `httpx/_client.py` or `httpx/_async_client.py` (verify dispatcher does **not** own cleanup per review feedback)

**Concrete file paths from repo tree:**
- Existing: `tests/client/test_async_client.py`, `tests/client/test_client.py`, `tests/conftest.py`, `tests/common.py`
- To be created: `tests/client/test_async_client_cancel.py`, `tests/client/test_client_cancel.py` (if sync applies)
- Implementation files: **Not visible in repo tree** (httpx source not included; paths are inferred as `httpx/_models.py`, `httpx/_async_pool.py` based on typical httpx structure. Implementer must locate actual files in the httpx source checkout or installed site-packages.)

**Notes:**
- The httpx source code is **not** present in the provided repo tree (`/home/pavc/Documents/Https-Demo`); only the test directory is visible. The implementer must ensure the httpx library source is available (e.g., cloned from httpx repo, or installed in editable mode, or modify in site-packages) to change `httpx/_models.py`, `httpx/_async_pool.py`, etc.
- If the httpx source is in a submodule or separate checkout, paths must be adjusted accordingly. The LLD will confirm actual file locations.
- The reproduction case in the ticket description (`asyncio.run(main())` with `httpx.AsyncClient` and `asyncio.wait_for`) must pass (no `RuntimeError`) after the fix.
- Sync client behavior: if httpx sync `Client` does not separately implement streaming reads (i.e., if sync and async share the same response model and cleanup logic), the sync parity requirement (AC-3) is automatically satisfied. Verify this in LLD.
- **Key architectural change (per review feedback):** Cleanup ownership is in `Response.aread()` and `Response.aiter_bytes()`, **not** in `AsyncClient._send_single_request()` or dispatcher. Do **not** add exception handling in the client dispatcher; the response methods own the cleanup via try/finally.

**Verification Approach:**

1. **Manual repro test:** Run the reproduction script from the ticket description. Before fix: `RuntimeError` raised. After fix: `TimeoutError` raised, no `RuntimeError`, clean exit.
2. **Unit test execution:** Run `tests/client/test_async_client_cancel.py` (4 tests). All must pass.
3. **Sync parity test execution:** Run `tests/client/test_client_cancel.py` (if applicable). All must pass.
4. **Existing test suite:** Run existing async client tests (`tests/client/test_async_client.py`) to ensure no regression in normal (non-cancelled) request flows.
5. **Code audit verification:** Confirm `Response.aclose()` idempotency (call twice, no error) and pool return shielding (trace code, verify shield or try/finally around pool release).

**Test count:** **Minimum 4 async tests** (cancel via `wait_for`, timeout via client config, response closed check, pool state check) + **4 sync tests if sync client exists** = **8 tests total** (or 4 if sync N/A). The LLD and final test implementation must include exactly these tests, with the same verification points, to satisfy AC-4.

**Acceptance Criteria (copied from requirements):**

- **AC-1:** Cancelling or timing out a streaming read always closes the response and returns the connection to the pool. *(Verified by tests 1, 2, 4)*
- **AC-2:** Closing `AsyncClient` after cancel does not raise the in-flight RuntimeError. *(Verified by tests 1, 2)*
- **AC-3:** Sync and async behaviour stay consistent for the same cancel path. *(Verified by sync mirror tests, or documented as N/A if sync client does not exist)*
- **AC-4:** Add regression tests for cancel and timeout while reading a drip/stream body. *(Verified by existence of `test_async_client_cancel.py` and `test_client_cancel.py` with required test cases)*

---

## Dependencies

**No inter-story dependencies.** This is a single-story fix.

**External dependencies:**
- httpx library source code must be available for modification (not visible in current repo tree; assumed to be in a separate checkout, installed in editable mode, or modified in site-packages).
- Test environment must support async tests (pytest with `pytest-asyncio` or equivalent; check `pyproject.toml` or `tests/conftest.py` for existing async test setup).

---

## Risks & Mitigations

*(Copied from HLD, filtered to those affecting this story)*

### Risk 1: Cancel raised after response.aclose() already called
**Impact on story:** Low (double-close must not error)  
**Mitigation in story:** Verify `response.aclose()` idempotency in step 3 (audit `_models.py`). Add a test case in `test_async_client_cancel.py` that calls `aclose()` twice and asserts no exception.

### Risk 2: Second cancel fires during aclose() execution
**Impact on story:** Medium (connection might not return to pool if pool release is interrupted)  
**Mitigation in story:** Step 3 includes adding `asyncio.shield()` around pool return in `aclose()`. Add test: cancel while `aclose()` is executing (requires timing or mock), verify pool state.

### Risk 3: Sync client diverges from async client in cleanup logic
**Impact on story:** High (AC-3 failure)  
**Mitigation in story:** Step 4 applies parallel fix to sync `read()` and `iter_bytes()`. Step 7 adds sync mirror tests. If sync client does not exist, document in LLD and mark AC-3 as N/A.

### Risk 4: Existing code assumes response auto-closes on context exit only
**Impact on story:** Medium (potential conflict with `__aexit__`)  
**Mitigation in story:** Step 1 includes auditing existing cleanup hooks (`__aexit__`, try/finally). Ensure new cleanup integrates (does not duplicate or conflict). Add test using `async with response` after cancel to verify no double-close error.

### Risk 5: httpx source not available in repo
**Impact on story:** Blocking  
**Mitigation in story:** LLD must confirm httpx source location. If not in repo, implementer must clone httpx or install editable. Document path resolution in LLD.

---

## Open Questions

*(Copied from HLD, with resolution path tied to this story)*

### Q1: Does the current Response.aclose() implementation shield the pool-return logic from cancellation?
**Resolution in this story:** Step 3 of implementation approach adds shielding to `httpx/_models.py` `Response.aclose()`. Trace pool return call, wrap in `asyncio.shield()`. Document findings in LLD. If already shielded, document and proceed; if not, add shielding as part of this story (Risk 2 mitigation).

### Q2: Are there existing try/finally blocks in Response.aread() or Response.aiter_bytes() that already attempt cleanup?
**Resolution in this story:** Step 1 and 2 include inspecting existing code for try/finally. LLD documents existing hooks. New cleanup integrates without duplication (verified by test in Risk 4 mitigation).

### Q3: Does the sync Client exist as a separate implementation?
**Resolution in this story:** LLD must check `httpx/_models.py` for sync `Response.read()` and `Response.iter_bytes()` methods. If sync methods exist with separate logic, apply parallel fix (step 4) and add sync tests (step 7). If sync methods do not exist or are thin wrappers, document in LLD and skip sync tests (mark AC-3 as satisfied by non-existence of divergent code).

### Q4: What test server/fixture is used for slow/drip endpoints?
**Resolution in this story:** Step 6 references `httpbin.org/drip` as default. LLD must check `tests/conftest.py` or `tests/common.py` for existing fixtures. If `httpbin` not available in CI, use mock server or pytest fixture that simulates slow response. Document test setup in LLD.

### Q5: Does the client dispatcher currently own any cancel handling that conflicts with response-owned cleanup?
**Resolution in this story:** Audit step (step 5 audit of `httpx/_client.py` or `httpx/_async_client.py`) confirms dispatcher does **not** catch `CancelledError`/`TimeoutError` and call `aclose()`. Per review feedback, dispatcher must **not** own cleanup. Document findings in LLD; if dispatcher has conflicting logic, remove it as part of this story.

---

## Estimation Notes

*(If enabled per `workflow.estimation`)*

**Complexity:** Moderate  
**Reasoning:**
- Core fix (steps 1-4) is ~40-60 lines of try/finally logic and shielding across Response methods in 1 file (`httpx/_models.py`).
- Audit (step 5) requires code tracing but no major changes expected.
- Test implementation (steps 6-7) is ~100-150 lines across 2 files.
- Risks are low-medium; mitigations are straightforward (idempotency check, shielding add, sync parity).
- No new public API, no schema changes, no cross-service coordination.

**T-shirt size:** M (Medium)  
**Story points:** 5 (assuming 1 point = ~1 day for a mid-level engineer; includes implementation, audit, testing, and risk mitigation)

---

## Validation Checklist

*(To be run by `bmad-ticket validate` before marking story as ready)*

- [ ] All 4 acceptance criteria (AC-1 through AC-4) are explicitly addressed in story description
- [ ] Test count matches verification approach (minimum 4 async + 4 sync if applicable)
- [ ] All file paths in "Allowed paths" are concrete or marked "see story LLD"
- [ ] All prerequisites are listed (none for this story)
- [ ] All risks from HLD affecting this story are listed with mitigations
- [ ] All open questions from HLD affecting this story have resolution paths
- [ ] Story covers all requirements from `requirements.md` (AC-1, AC-2, AC-3, AC-4)
- [ ] No new acceptance criteria invented beyond ticket and requirements
- [ ] Description outcomes are behavioral (e.g., "response closes, no RuntimeError"), not language-specific syntax
- [ ] Repo key matches project context (`Https-test`)

---

**End of Breakdown.**
