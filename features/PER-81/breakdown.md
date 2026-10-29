# Breakdown — Close response stream when async request is cancelled mid-read

**Ticket:** PER-81  
**Type:** Bug  
**Epic:** (standalone)  
**Date:** 2025-01-23

---

## Overview

This breakdown converts the HLD architectural decisions and requirements into a single implementable story. The story addresses the core issue: when an `AsyncClient` request is cancelled or times out during response streaming, the response stream is not closed, leaving the connection marked as in-flight and causing a `RuntimeError` when the client exits.

The fix implements response cleanup ownership at the request dispatcher level, ensures sync/async consistency, and adds comprehensive regression tests covering cancel and timeout scenarios.

---

## Stories

### Story 1: Implement response cleanup on async cancel/timeout with sync parity and tests

**Repo:** `Https-test`

**Description:**

When an `AsyncClient` request is cancelled (via `asyncio.wait_for` timeout or explicit cancellation) or times out (via client timeout) while the response body is streaming, the response stream must be closed and the connection returned to the pool before the exception propagates. The sync `Client` must exhibit identical cleanup behavior. This story implements the cleanup logic in both async and sync request dispatchers, verifies the response `aclose()`/`close()` implementation is idempotent and cancel-safe, and adds regression tests for all cancel/timeout scenarios.

**Covers:**
- **AC-1:** Response closure on cancel
- **AC-2:** Response closure on timeout  
- **AC-3:** Sync/async consistency
- **AC-4:** Regression test coverage

**Prerequisites:** None (standalone bug fix)

**Depends on (after):** None

**Architectural Decisions Applied:**
- **AD-1:** Response cleanup ownership on async cancel/timeout — dispatcher catches `CancelledError`/`TimeoutError`, calls `response.aclose()`, re-raises
- **AD-2:** Response stream closure mechanism — `response.aclose()` is the single entry point for cleanup
- **AD-3:** Sync/async consistency — sync `Client` mirrors async cleanup logic
- **AD-4:** Cancel detection point — cancel/timeout detected at request dispatcher layer, not in pool/transport

**Implementation approach:**

1. **Async cleanup (primary fix):**
   - Locate the async request dispatcher method in `httpx/_async_client.py` or `httpx/_client.py` (inferred as `_send_single_request` or equivalent) that orchestrates the request and awaits the response body read.
   - Wrap the response-reading operation (e.g., awaiting `response.aiter_bytes()`, `response.aread()`, or the method that consumes the body) in a `try/except` block that catches `asyncio.CancelledError` and `asyncio.TimeoutError`.
   - In the exception handler, call `await response.aclose()` before re-raising the exception.
   - Ensure the fix does not conflict with existing `try/finally` or context manager cleanup (audit for existing `response.aclose()` calls and `Response.__aexit__` behavior).

2. **Sync cleanup (parity fix):**
   - Locate the sync request dispatcher in `httpx/_client.py` (if separate from async, or the sync equivalent method).
   - Apply parallel logic: catch sync interrupt exceptions (e.g., `KeyboardInterrupt`, or whatever the sync equivalent of cancel/timeout is in the httpx sync client), call `response.close()`, re-raise.
   - Verify sync and async cleanup structures are identical (same exception handling pattern, same cleanup call, same re-raise).

3. **Response idempotency and cancel-safety audit:**
   - Open `httpx/_models.py` and locate the `Response.aclose()` and `Response.close()` implementations.
   - Verify `aclose()` is idempotent (can be called multiple times without error; typically checked via an `_is_closed` or `_closed` flag).
   - Trace the pool return logic within `aclose()` (likely a call to `self._pool.return_connection(self._connection)` or similar).
   - Check if the pool return is wrapped in `asyncio.shield()` or a `try/finally` to ensure it completes even if `aclose()` is cancelled mid-execution (Risk 2 mitigation).
   - If not shielded, add shielding around the pool return operation to guarantee the connection is released.

4. **Connection pool audit (verify, no change expected):**
   - Open `httpx/_async_pool.py` (or equivalent pool module).
   - Verify the pool's connection release logic correctly decrements the in-flight count when `response.aclose()` returns a connection.
   - Confirm the pool does **not** attempt to detect cancel/timeout itself (per AD-4); it should only receive release signals from response objects.
   - Document findings; no code change expected unless a discrepancy is found.

5. **Regression tests (async):**
   - Create `tests/client/test_async_client_cancel.py`.
   - Implement the following tests:
     - **`test_cancel_during_streaming_read`**: Use `asyncio.wait_for(client.get(url), timeout=3)` with a slow endpoint (e.g., `https://httpbin.org/drip?duration=5`). Assert `asyncio.TimeoutError` is raised and `async with client` exits without `RuntimeError`.
     - **`test_client_timeout_during_streaming_read`**: Configure `AsyncClient(timeout=3)`, GET a slow endpoint, let the client timeout fire (not `asyncio.wait_for`). Assert `httpx.TimeoutException` (or equivalent) is raised and client exits cleanly.
     - **`test_response_closed_after_cancel`**: Cancel during streaming, catch exception, assert `response.is_closed` (or equivalent property) is `True`.
     - **`test_connection_returned_to_pool_after_cancel`**: If pool exposes in-flight count (e.g., `client._pool._connections` or similar), verify count returns to 0 after cancel. If not exposed, this test is satisfied by the absence of `RuntimeError` in test 1.
   - Use a test fixture or mock server that provides a slow/drip endpoint if `httpbin.org` is not available in CI (check existing test setup in `tests/conftest.py` or `tests/common.py`).

6. **Regression tests (sync, if applicable):**
   - If the sync `Client` exists as a separate implementation (check `httpx/_client.py` for sync methods), create `tests/client/test_client_cancel.py`.
   - Mirror the async tests above, adapted for sync:
     - Use `threading.Timer` or signal-based timeout to simulate interrupt.
     - Replace `asyncio.TimeoutError` with sync equivalents (e.g., `TimeoutError` or `KeyboardInterrupt` depending on httpx sync timeout mechanism).
     - Replace `async with` with `with` and `await` with blocking calls.
   - If sync client does not exist or is deprecated, document this in Notes and skip sync tests (but still verify AC-3 is not violated by checking if sync code paths exist at all).

**Allowed paths:**

Primary changes (must exist or be located):
- `httpx/_async_client.py` or `httpx/_client.py` (async request dispatcher) — add try/except around response read, call `await response.aclose()`
- `httpx/_client.py` (sync request dispatcher, if separate) — add try/except around response read, call `response.close()`
- `httpx/_models.py` (Response class) — audit `aclose()`/`close()` for idempotency and shielding; add shielding if missing

Test files (created):
- `tests/client/test_async_client_cancel.py` (new file, 4 tests minimum)
- `tests/client/test_client_cancel.py` (new file if sync client exists, 4 tests mirroring async)

Audit-only (read, no expected changes):
- `httpx/_async_pool.py` (verify pool release logic)
- `tests/client/test_async_client.py` (check for conflicting existing tests)
- `httpx/_transports/` or `httpx/_backends/` (if exists, verify cancel not detected at transport layer)

**Concrete file paths from repo tree:**
- Existing: `tests/client/test_async_client.py`, `tests/client/test_client.py`, `tests/conftest.py`, `tests/common.py`
- To be created: `tests/client/test_async_client_cancel.py`, `tests/client/test_client_cancel.py` (if sync applies)
- Implementation files: **Not visible in repo tree** (httpx source not included; paths are inferred as `httpx/_async_client.py`, `httpx/_models.py`, `httpx/_async_pool.py` based on typical httpx structure. Implementer must locate actual files in the httpx source checkout.)

**Notes:**
- The httpx source code is **not** present in the provided repo tree (`/home/pavc/Documents/Https-Demo`); only the test directory is visible. The implementer must ensure the httpx library source is available (e.g., cloned from httpx repo or installed in editable mode) to modify `httpx/_async_client.py`, `httpx/_models.py`, etc.
- If the httpx source is in a submodule or separate checkout, paths must be adjusted accordingly. The LLD will confirm actual file locations.
- The reproduction case in the ticket description (`asyncio.run(main())` with `httpx.AsyncClient` and `asyncio.wait_for`) must pass (no `RuntimeError`) after the fix.
- Sync client behavior: if httpx sync `Client` does not separately implement streaming reads (i.e., if sync and async share the same dispatcher logic), the sync parity requirement (AC-3) is automatically satisfied. Verify this in LLD.

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
- httpx library source code must be available for modification (not visible in current repo tree; assumed to be in a separate checkout or editable install).
- Test environment must support async tests (pytest with `pytest-asyncio` or equivalent; check `pyproject.toml` or `tests/conftest.py` for existing async test setup).

---

## Risks & Mitigations

*(Copied from HLD, filtered to those affecting this story)*

### Risk 1: Cancel raised after response.aclose() already called
**Impact on story:** Low (double-close must not error)  
**Mitigation in story:** Verify `response.aclose()` idempotency in step 3 (audit `_models.py`). Add a test case in `test_async_client_cancel.py` that calls `aclose()` twice and asserts no exception.

### Risk 2: Timeout fires in the middle of aclose() itself
**Impact on story:** Medium (connection might not return to pool)  
**Mitigation in story:** Step 3 includes auditing pool return shielding in `aclose()`. If not shielded, add `asyncio.shield()` around pool return. Add test: cancel while `aclose()` is executing (requires timing or mock), verify pool state.

### Risk 3: Sync client diverges from async client in cleanup logic
**Impact on story:** High (AC-3 failure)  
**Mitigation in story:** Step 2 applies parallel fix to sync dispatcher. Step 6 adds sync mirror tests. If sync client does not exist, document in LLD and mark AC-3 as N/A.

### Risk 4: Existing code assumes response auto-closes on context exit
**Impact on story:** Medium (potential conflict with `__aexit__`)  
**Mitigation in story:** Step 1 includes auditing existing cleanup hooks (`__aexit__`, try/finally). Ensure new cleanup integrates (does not duplicate or conflict). Add test using `async with response` after cancel to verify no double-close error.

### Risk 5: httpx source not available in repo
**Impact on story:** Blocking  
**Mitigation in story:** LLD must confirm httpx source location. If not in repo, implementer must clone httpx or install editable. Document path resolution in LLD.

---

## Open Questions

*(Copied from HLD, with resolution path tied to this story)*

### Q1: Does the current `AsyncResponse` implementation shield the pool-return logic in `aclose()` from cancellation?
**Resolution in this story:** Step 3 of implementation approach audits `httpx/_models.py` `Response.aclose()`. Trace pool return call, check for `asyncio.shield()` or try/finally. Document findings in LLD. If not shielded, add shielding as part of this story (Risk 2 mitigation).

### Q2: Where exactly is the cancel/timeout detected in the current code?
**Resolution in this story:** Step 1 requires locating the async request dispatcher. LLD must trace call stack from `AsyncClient.get()` to response body read. Confirm dispatcher is correct layer per AD-4. Document actual detection point in LLD.

### Q3: Are there existing `try/finally` blocks around response operations that already attempt cleanup?
**Resolution in this story:** Step 1 includes grepping for existing cleanup. LLD documents existing hooks. New cleanup integrates without duplication (verified by test in Risk 4 mitigation).

### Q4: Does the sync Client exist as a separate implementation?
**Resolution in this story:** LLD must check `httpx/_client.py` for sync request dispatcher. If sync client does not exist or is deprecated, document in LLD and skip sync tests (mark AC-3 as satisfied by non-existence of divergent code).

### Q5: What test server/fixture is used for slow/drip endpoints?
**Resolution in this story:** Step 5 references `httpbin.org/drip` as default. LLD must check `tests/conftest.py` or `tests/common.py` for existing fixtures. If `httpbin` not available in CI, use mock server or pytest fixture that simulates slow response. Document test setup in LLD.

---

## Estimation Notes

*(If enabled per `workflow.estimation`)*

**Complexity:** Moderate  
**Reasoning:**
- Core fix (steps 1-2) is ~20-30 lines of exception handling logic across 1-2 files.
- Audit (steps 3-4) requires code tracing but no major changes expected.
- Test implementation (steps 5-6) is ~100-150 lines across 2 files.
- Risks are low-medium; mitigations are straightforward (idempotency check, shielding add).
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
