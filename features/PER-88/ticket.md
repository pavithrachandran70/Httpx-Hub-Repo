---
key: PER-88
type: story
status: todo
blocks: []
blocked_by: []
labels: []
repos: []
summary: Client.stream ignores auth from Client constructor
---

 Auth set on `httpx.Client(auth=(user, password))` is used by `get`/`post`, but not by `Client.stream`, which returns 401 unless auth is passed again on the call.

Repro

```
cli = httpx.Client(auth=("user", "password"))
cli.get(url).raise_for_status()          # OK

with cli.stream("GET", url) as resp:
    resp.raise_for_status()              # 401

with cli.stream("GET", url, auth=cli.auth) as resp:
    resp.raise_for_status()              # OK
```

Acceptance criteria

* `Client.stream` and `AsyncClient.stream` use the same auth resolution as `request`/`get`/`post`.
* Per-call `auth=` still overrides client-level auth.
* Add sync and async regression tests.
