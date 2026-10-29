**AC gap: Async `aiter_bytes()` cleanup guard is overly narrow**

The `finally` block at line 1026 checks `isinstance(self.stream, AsyncByteStream)` before calling `await self.aclose()`. Same issue as `aread()`.

**AC-1 / AC-2 violation:** An async streaming iteration cancelled or timed out may skip cleanup if the stream type is wrong.

**Guard snippet:**
```python
finally:
    await self.aclose()  # Unconditional; aclose() is idempotent
```
