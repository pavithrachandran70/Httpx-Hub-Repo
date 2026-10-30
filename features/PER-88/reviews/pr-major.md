**AC-4 PARTIAL: Tests verify the fix, but the fix itself is missing**

AC-4 requires "new regression tests in `tests/client/test_auth.py` verify" the fix — these tests are present and correctly structured:

- ✅ `test_client_stream_inherits_auth` — sync client-level auth inheritance
- ✅ `test_async_client_stream_inherits_auth` — async client-level auth inheritance  
- ✅ `test_client_stream_auth_override` — sync per-call override
- ✅ `test_async_client_stream_auth_override` — async per-call override
- ✅ `test_client_stream_auth_parity_with_get` — sync parity across methods
- ✅ `test_async_client_stream_auth_parity_with_get` — async parity across methods

However, **the actual implementation changes to make these tests pass are absent from the diff**. The tests are well-designed but cannot pass without the corresponding code fix in the client implementation files.
