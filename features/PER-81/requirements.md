# Requirements — Close response stream when async request is cancelled mid-read

**Ticket:** PER-81  
**Type:** Bug  
**Epic:** (standalone)

## Problem Statement

When an `AsyncClient` request is cancelled or times out while the response body is still streaming, the response stream is not properly closed. This causes the connection to remain marked as in-flight, and when the client context manager exits, a `RuntimeError` is raised reporting that the connection pool was closed while HTTP requests/responses were still active.

## Acceptance Criteria

### AC-1: Response closure on cancel

**Given** an `AsyncClient` with an active streaming response  
**When** the request is cancelled via `asyncio.wait_for` timeout or explicit cancellation  
**Then** the response stream is closed and the connection is returned to the pool  
**And** no in-flight connection error is raised when the client exits

### AC-2: Response closure on timeout

**Given** an `AsyncClient` with an active streaming response  
**When** the request times out via the client's configured timeout  
**Then** the response stream is closed and the connection is returned to the pool  
**And** no in-flight connection error is raised when the client exits

### AC-3: Sync/async consistency

**Given** equivalent cancel or timeout scenarios in sync and async clients  
**When** a streaming read is interrupted  
**Then** both sync `Client` and `AsyncClient` exhibit the same cleanup behavior  
**And** both properly close the response and return the connection to the pool

### AC-4: Regression test coverage

**Given** the fix is implemented  
**When** regression tests are executed  
**Then** tests exist that verify cancel and timeout during streaming reads  
**And** tests confirm no `RuntimeError` about in-flight connections when client closes  
**And** tests cover both `asyncio.wait_for` cancellation and client-timeout paths

## Success Criteria

1. The reproduction case in the ticket description runs without raising `RuntimeError`.
2. Connections are properly released back to the pool after cancel or timeout.
3. No behavioral regression in normal (non-cancelled) request flows.
4. Test suite includes explicit coverage for the cancel and timeout scenarios.

## Out of Scope

- Changes to the connection pool's capacity or reuse strategy unrelated to cleanup on cancel
- Performance optimizations beyond fixing the resource leak
- Timeout configuration or policy changes
- Behavior of other client types (e.g., `Client` sync changes beyond consistency with async)

## Notes

- The core issue is response lifecycle ownership during exceptional flow (cancel/timeout), not during normal completion.
- The fix must ensure the response's underlying stream and connection are released even when the coroutine reading the response is cancelled.
- Sync and async code paths should mirror each other to avoid divergent behavior on the same failure mode.
