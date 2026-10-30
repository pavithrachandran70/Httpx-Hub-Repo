# Requirements — PER-83

**Epic:** (none — backlog bug)  
**Story:** Close response stream when async request is cancelled mid-read  
**Created:** 2025-01-24  
**Status:** draft

---

## Overview

When an `AsyncClient` request is cancelled or timed out while the response body is still streaming, the response object is not closed, leaving the connection in an in-flight state. Exiting the client then raises `RuntimeError: The connection pool was closed while 1 HTTP requests/responses were still in-flight.`

This story ensures that cancellation or timeout during a streaming read always closes the response and returns the connection to the pool, preventing the resource leak and the in-flight error on client cleanup.

---

## Acceptance Criteria

### AC1: Cancel during stream closes response and returns connection

**Given** an `AsyncClient` with a response body streaming from a slow endpoint  
**When** the request is cancelled via `asyncio.wait_for` timeout or explicit task cancellation before the stream completes  
**Then** the response is closed immediately and the connection is returned to the pool without raising an in-flight error on client exit

---

### AC2: Timeout during stream closes response and returns connection

**Given** an `AsyncClient` with a timeout configured and a response body streaming from a slow endpoint  
**When** the client timeout fires while the stream is still reading  
**Then** the response is closed immediately and the connection is returned to the pool without raising an in-flight error on client exit

---

### AC3: Sync and async cancel behavior is consistent

**Given** both sync `Client` and `AsyncClient` with streaming responses  
**When** a request is cancelled or timed out mid-read in either client  
**Then** both clients close the response and return the connection to the pool in the same manner, with no in-flight connection error on cleanup

---

### AC4: Regression tests cover cancel and timeout during streaming read

**Given** the httpx test suite  
**When** tests are run  
**Then** regression tests exist covering:
- Cancellation via `asyncio.wait_for` timeout while reading a drip/stream endpoint
- Timeout via client timeout setting while reading a drip/stream endpoint
- Verification that no in-flight `RuntimeError` is raised on client exit after cancel/timeout

---

## Reproduction Case

```python
import asyncio
import httpx

async def main():
    url = "https://httpbin.org/drip?delay=0&duration=5"
    async with httpx.AsyncClient(timeout=10, trust_env=False) as client:
        try:
            await asyncio.wait_for(client.get(url), 3)
        except Exception as ex:
            print(type(ex), repr(ex))

asyncio.run(main())
```

Expected: No `RuntimeError` about in-flight connections on client exit.  
Actual (before fix): `RuntimeError: The connection pool was closed while 1 HTTP requests/responses were still in-flight.`

---

## Notes

- The repro uses `httpbin.org/drip` to simulate a slow-streaming response; tests should use a local mock or fixture with similar behavior.
- The fix must ensure response cleanup on both `asyncio.CancelledError` and timeout paths.
- Sync client behavior for the same scenario should be verified for consistency (AC3).

---

## Open Questions

(none)

---

## References

- Ticket: PER-83
- Repo: `Https-test` (https://github.com/pavithrachandran70/httpx-demo)
- Base branch: `master`
