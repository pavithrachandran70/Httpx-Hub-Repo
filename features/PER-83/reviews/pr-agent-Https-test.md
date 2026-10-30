```json
[
  {
    "path": "httpx/_client.py",
    "line": 935,
    "body": "**AC1/AC2 gap:** The `finally` block only closes the response when `stream=False`. When `stream=True`, the response is left open for the caller to read, but if the caller cancels mid-read (e.g., `async with client.stream(...) as response: async for chunk in response.aiter_bytes(): ...` wrapped in `asyncio.wait_for`), the response is never closed. The LLD states \"response context manager or internal `_send_handling_redirects()`\" as the fix location, but the context manager (`stream()`) has no equivalent cleanup logic. **Guard:** Add a `finally` block inside `AsyncClient.stream()` (the async context manager) to close the response on cancellation during body iteration, or document that the caller must handle cancellation cleanup when using `stream()`."
  },
  {
    "path": "httpx/_client.py",
    "line": 919,
    "body": "**AC1/AC2 edge case:** `finally` runs even when `response` is assigned but `response.read()` or `response.aread()` raises before completion. If `read()` raises a non-cancellation exception (e.g., `httpx.ReadError`), the response is still closed by the `finally` block, which is correct. However, the comment `# return the connection to the pool before a timeout or any other exception propagates` implies this is only for cancellation/timeout, but the code closes on *all* exceptions when `stream=False`. **Clarification:** The behavior is correct (close on any exception when not streaming), but the comment should say \"before *any* exception propagates\" to match the code, or the code should check `isinstance(exc, (asyncio.CancelledError, asyncio.TimeoutError, httpx.ReadTimeout))` if only cancellation/timeout cleanup is intended."
  },
  {
    "path": "httpx/_client.py",
    "line": 1658,
    "body": "**AC1 gap (async):** Same issue as the sync path: when `stream=True`, the `finally` block skips cleanup, leaving the response open. If the caller uses `client.stream()` and cancels mid-read, the response is not closed here. The async `stream()` method (context manager) must handle cleanup on cancellation, but the diff shows no changes to that method. **Guard:** Review `AsyncClient.stream()` (the `@asynccontextmanager` or equivalent) and ensure it has a `finally` block calling `await response.aclose()` on cancellation during the context body."
  },
  {
    "path": "tests/client/test_async_client.py",
    "line": 503,
    "body": "**AC1 verification gap:** Test `test_async_cancel_during_body_iteration_closes_response` uses `client.stream()` and cancels during `aiter_bytes()`. The test expects `seen[-1].is_closed` to be true, but the `_client.py` fix only closes responses when `stream=False`. If `stream()` does not have its own cleanup logic (not shown in the diff), this test will **fail**. **Evidence:** The diff adds no cleanup to `stream()`, only to `send(..., stream=False)`. AC1 says \"response is closed immediately\" on cancel; the test verifies that, but the code does not implement it for streaming responses."
  },
  {
    "path": "tests/client/test_async_client.py",
    "line": 379,
    "body": "**AC4 gap:** Tests cover `asyncio.wait_for` cancellation (AC1) and client timeout (AC2) for async, and sync timeout (AC3), but no test explicitly verifies that the `RuntimeError: The connection pool was closed while 1 HTTP requests/responses were still in-flight` does **not** occur on client exit. The tests check `_inflight_requests(client) == []` before the `async with` block exits, but they do not assert that `client.__aexit__()` completes without raising. **Guard:** Add an assertion after each test's `async with` block exits (e.g., `# __aexit__ completed without RuntimeError`) or wrap the block in a `try/except RuntimeError` that fails if the error is raised, to explicitly verify AC1/AC2's \"without raising an in-flight error on client exit\" clause."
  },
  {
    "path": "tests/client/test_client.py",
    "line": 497,
    "body": "**AC3 verification:** Test `test_sync_timeout_during_stream_closes_response` verifies that the sync client closes the response on timeout and returns the connection, matching async behavior. The test checks `seen[-1].is_closed` and `list(pool._requests) == []`. However, AC3 says \"both clients close the response and return the connection **in the same manner**\". The sync path in `_client.py` uses `response.close()` in the `finally` block, but the test does not verify that the `with httpx.Client(...)` block exits without raising the in-flight `RuntimeError`, only that the connection is returned before exit. **Guard:** Same as async tests — add an explicit assertion or comment that `__exit__` completes without `RuntimeError`."
  },
  {
    "path": "httpx/_client.py",
    "line": 935,
    "body": "**Potential bug:** `if not stream and response is not None and not response.is_closed:` — the condition `not response.is_closed` is checked, but `response.is_closed` may not be defined or may raise `AttributeError` if `response` is a partial or malformed object (e.g., if `_send_handling_auth` raises before fully constructing the `Response`). The comment says \"`response` stays None when sending fails before a response exists\", which handles the `None` case, but does not guard against a response object that exists but is incomplete. **Guard:** Wrap the check in `hasattr(response, 'is_closed') and not response.is_closed`, or catch `AttributeError` when calling `response.close()`, to handle edge cases where response construction fails partway."
  },
  {
    "path": ".gitignore",
    "line": 13,
    "body": "**Process:** `.gitignore` adds ADLC/Cursor tooling entries (`__pycache__`, `*.pyc`, `.pytest_cache`, `.mypy_cache`, `.ruff_cache`, `.env`, `*.egg-info`, `.venv`, `venv`). These are standard Python ignores, but several are duplicates of existing entries at the top of the file (lines 1–12 already ignore `__pycache__/`, `*.py[cod]`, `.venv/`, `venv*/`). The comment \"injected at Dev time; not product code\" suggests this is tooling scaffolding. **Clarification:** The duplicates do not break functionality (git ignores duplicate patterns), but they clutter the file. Consider removing duplicates or merging into a single \"Development tooling\" section if this is intentional."
  }
]
```
