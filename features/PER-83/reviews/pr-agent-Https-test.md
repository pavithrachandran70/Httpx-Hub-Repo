```json
[
  {
    "path": "httpx/_client.py",
    "line": 930,
    "body": "**AC1/AC2 violation (async):** The `finally` block only closes the response when `stream=False`. When `stream=True`, the response is returned open to the caller, but there is no protection if the caller's code (e.g., iterating `aiter_bytes()`) is cancelled mid-read. The async client's `stream()` context manager should wrap its yield in a similar cleanup to ensure cancellation during iteration closes the response. The test `test_async_cancel_during_body_iteration_closes_response` covers this scenario but the implementation does not guard it—`stream()` relies on the caller closing the response, which cancellation interrupts."
  },
  {
    "path": "httpx/_client.py",
    "line": 922,
    "body": "**AC1/AC2 violation (sync):** The sync client's `finally` block only closes the response when `stream=False`. When `stream=True`, the response is returned open. Unlike async, the sync client does not support mid-iteration cancellation via `asyncio.CancelledError`, but it must still handle timeouts during `response.iter_bytes()`. The test `test_sync_timeout_during_stream_closes_response` does not cover iteration (only `client.get(url)` with `stream=False`), so AC3 parity is incomplete: there is no sync equivalent of `test_async_cancel_during_body_iteration_closes_response` to verify sync streaming iteration cleanup."
  },
  {
    "path": "tests/client/test_async_client.py",
    "line": 519,
    "body": "**AC3 gap:** The async tests use `anyio_backend_name` to support both asyncio and trio, but `test_async_cancel_during_body_iteration_closes_response` does not parameterize on backend—it only runs on the default backend. The other two async tests call `_await_with_deadline(..., anyio_backend_name)` correctly, but this test does not pass `anyio_backend_name` to `_await_with_deadline` on line 543. Either add the parameter or remove `anyio_backend_name` from the other tests' signatures if trio is not in scope."
  },
  {
    "path": "tests/client/test_client.py",
    "line": 497,
    "body": "**AC3 incomplete:** `test_sync_timeout_during_stream_closes_response` calls `client.get(url)` with `stream=False` (the default), so it tests the same path as the async non-streaming test. AC3 requires sync/async parity for streaming reads; a sync equivalent of `test_async_cancel_during_body_iteration_closes_response` is missing. Either add a test where `client.stream('GET', url)` times out during `response.iter_bytes()`, or document that sync streaming iteration timeouts are out of scope."
  },
  {
    "path": "tests/conftest.py",
    "line": 275,
    "body": "**Process note:** The `serve_in_thread` timeout guards (lines 275–283 and 289–292) are defensive additions not mentioned in the story or AC4. They prevent test hangs if uvicorn fails to start, which is good hygiene but unrelated to the response-cleanup fix. If these changes are intentional cleanup, they belong in a separate commit or PR; if they were added to work around a uvicorn startup issue in the test environment, document that in a comment."
  },
  {
    "path": ".gitignore",
    "line": 13,
    "body": "**Process note:** The `.gitignore` additions (lines 13–27) include ADLC/Cursor tooling artifacts and duplicate several patterns already present (lines 5–12). The comment 'injected at Dev time; not product code' suggests these are local development artifacts. If they are required for this PR's test runs, explain why in the PR description; otherwise remove them—`.gitignore` changes unrelated to the story dilute the PR's focus."
  }
]
```
