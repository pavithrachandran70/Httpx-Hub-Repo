```json
[
  {
    "path": "tests/client/test_auth.py",
    "line": 793,
    "body": "**AC-1 VIOLATION: Missing actual fix implementation**\n\nThe diff adds comprehensive test coverage for `Client.stream()` and `AsyncClient.stream()` auth inheritance, but **does not include the actual code changes** to the `Client` and `AsyncClient` classes that would make these tests pass.\n\n**Expected per story Technical Specification:**\n- Modifications to `Client.stream()` method to apply auth resolution logic\n- Modifications to `AsyncClient.stream()` method to apply auth resolution logic\n\n**Impact:** Tests will fail on PR merge because the underlying bug is not fixed. The diff is test-only."
  },
  {
    "path": "tests/client/test_auth.py",
    "line": 793,
    "body": "**AC-4 PARTIAL: Tests verify the fix, but the fix itself is missing**\n\nAC-4 requires \"new regression tests in `tests/client/test_auth.py` verify\" the fix — these tests are present and correctly structured:\n\n- ✅ `test_client_stream_inherits_auth` — sync client-level auth inheritance\n- ✅ `test_async_client_stream_inherits_auth` — async client-level auth inheritance  \n- ✅ `test_client_stream_auth_override` — sync per-call override\n- ✅ `test_async_client_stream_auth_override` — async per-call override\n- ✅ `test_client_stream_auth_parity_with_get` — sync parity across methods\n- ✅ `test_async_client_stream_auth_parity_with_get` — async parity across methods\n\nHowever, **the actual implementation changes to make these tests pass are absent from the diff**. The tests are well-designed but cannot pass without the corresponding code fix in the client implementation files."
  }
]
```
