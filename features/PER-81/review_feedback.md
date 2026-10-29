# Review feedback (architect_review, plan v3)

- HLD AD-1 and AD-4 are wrong. Do not catch CancelledError or TimeoutError in AsyncClient._send_single_request and then call aclose().

Change the design:
- Response.aread and Response.aiter_bytes own the cleanup, not the client dispatcher.
- Wrap the body read in try/finally and always call aclose() on the way out, including cancel and timeout.
- Shield the pool release inside aclose() with asyncio.shield so a second cancel during close still returns the connection.
- Update the HLD, the file touch list, and the tests to match this. The client dispatcher must not own this path.
