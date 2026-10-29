---
key: PER-81
type: story
status: todo
blocks: []
blocked_by: []
labels: []
repos: []
summary: Close response stream when async request is cancelled mid-read
---

Description: When an `AsyncClient` request is cancelled or timed out while the response body is still streaming, the response is not closed. Exiting the client then raises:

`RuntimeError: The connection pool was closed while 1 HTTP requests/responses were still in-flight.`

Repro

```
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

Acceptance criteria

* Cancelling or timing out a streaming read always closes the response and returns the connection to the pool.
* Closing `AsyncClient` after cancel does not raise the in-flight RuntimeError.
* Sync and async behaviour stay consistent for the same cancel path.
* Add regression tests for cancel and timeout while reading a drip/stream body.
