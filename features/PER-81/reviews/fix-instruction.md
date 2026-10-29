Fix these forge PR review comments (auto-route):

### Https-test comment 4410687814
**Test artifact gap: `_inflight_requests()` helper is fragile and may break in production**

The helper at line 22 accesses `client._transport._pool._requests` directly:
```python
pool = getattr(client._transport, "_pool")
return len(pool._requests)
```

**Problems:**
1. **Private API dependency:** `_transport`, `_pool`, and `_requests` are all private (`_` prefix). The test will break if httpcore refactors pool internals.
2. **No fallback:** If `_pool` or `_requests` does not exist (e.g., different transport implementation, or httpcore version change), `getattr` returns `None` and `len(pool._requests)` raises `AttributeError`.
3. **AC-4 gap:** Tests must confirm "no `RuntimeError` about in-flight connections when client closes" (per AC-4). The current approach infers that from pool state, but does not actually verify the absence of the error.

**Guard snippet (robust helper):**
```python
def _inflight_requests(client: httpx.AsyncClient) -> int:
    """Requests still checked out of the httpcore connection pool."""
    try:
        pool = getattr(client._transport, "_pool", None)
        if pool is None:
            return 0  # Transport has no pool; assume clean
        requests = getattr(pool, "_requests", [])
        return len(requests)
    except (AttributeError, TypeError):
        return 0  # Pool structure changed; assume clean (test will fail elsewhere if broken)
```

**Better approach:** Add an explicit test that wraps `async with client` in a try/except and asserts no `RuntimeError` is raised, rather than inferring it from pool state.

### Https-test comment 4410687821
**AC-4 gap: No explicit test for "no RuntimeError when client exits"**

AC-4 requires: "tests confirm no `RuntimeError` about in-flight connections when client closes". The current tests infer this from `_inflight_requests(client) == 0`, but do not explicitly verify the absence of the error.

**Problem:** If the pool's in-flight count is wrong (e.g., decremented on cancel but the connection is still held elsewhere), the test passes but the client exit still raises `RuntimeError`.

**Guard snippet (explicit error check):**
```python
@pytest.mark.anyio
async def test_no_runtime_error_on_client_exit_after_cancel(server):
    """
    The client context manager exits cleanly after a cancelled streaming read,
    without raising RuntimeError about in-flight connections.
    """
    url = server.url.copy_with(path="/slow_response")

    try:
        async with httpx.AsyncClient() as client:
            with pytest.raises(asyncio.TimeoutError):
                await asyncio.wait_for(client.get(url), timeout=0.2)
        # If we reach here, no RuntimeError was raised during __aexit__
    except RuntimeError as e:
        if "in-flight" in str(e).lower() or "active" in str(e).lower():
            pytest.fail(f"Client raised RuntimeError on exit after cancel: {e}")
        raise  # Re-raise if it's a different RuntimeError
```

Add this test to explicitly satisfy AC-4's "no RuntimeError" clause.

### Https-test comment 4410687829
**Test artifact gap: Sync `_inflight_requests()` has same fragility as async version**

Same issue as async test helper: accesses `_transport._pool._requests` directly with no fallback. See async test finding for details.

**Guard snippet:** Same robust fallback as async version.

### Https-test comment 4410687833
**AC-4 gap: No explicit test for "no RuntimeError when sync client exits"**

AC-3 requires sync/async parity, and AC-4 requires tests confirm no in-flight error on client close. The sync tests have the same gap as async: they infer clean exit from pool state rather than verifying the absence of `RuntimeError`.

**Guard snippet (explicit error check):**
```python
def test_no_runtime_error_on_sync_client_exit_after_cancel(server):
    """
    The sync client context manager exits cleanly after a timed-out streaming read,
    without raising RuntimeError about in-flight connections.
    """
    url = server.url.copy_with(path="/slow_response")
    timeout = httpx.Timeout(5.0, read=0.2)

    try:
        with httpx.Client(timeout=timeout) as client:
            with pytest.raises(httpx.TimeoutException):
                client.get(url)
        # If we reach here, no RuntimeError was raised during __exit__
    except RuntimeError as e:
        if "in-flight" in str(e).lower() or "active" in str(e).lower():
            pytest.fail(f"Client raised RuntimeError on exit after timeout: {e}")
        raise  # Re-raise if it's a different RuntimeError
```

Add this test to satisfy AC-3/AC-4 parity.

### Https-test comment 4410687863
**Test completeness gap: Missing test for `asyncio.CancelledError` path**

The test suite covers `asyncio.TimeoutError` (via `asyncio.wait_for` timeout) and `httpx.TimeoutException` (via client timeout), but does not cover explicit cancellation with `asyncio.CancelledError`.

**AC-1 says:** "the request is cancelled via `asyncio.wait_for` timeout **or explicit cancellation**" (emphasis added). `asyncio.wait_for` timeout raises `TimeoutError`, not `CancelledError`. Explicit cancellation means calling `task.cancel()` on the request task.

**Missing test:**
```python
@pytest.mark.anyio
async def test_explicit_cancel_during_streaming_read(server):
    """
    Explicit task.cancel() during a streaming read closes the response
    and returns the connection before the client exits.
    """
    url = server.url.copy_with(path="/slow_response")

    async def fetch():
        async with httpx.AsyncClient() as client:
            await client.get(url)

    task = asyncio.create_task(fetch())
    await asyncio.sleep(0.1)  # Let request start
    task.cancel()
    
    with pytest.raises(asyncio.CancelledError):
        await task
    
    # Client already exited in fetch(); if it raised RuntimeError, test fails
```

Add this test to fully cover AC-1's "explicit cancellation" clause.
