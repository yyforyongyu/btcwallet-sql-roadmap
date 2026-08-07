# Task 325 — post-delivery API and test cleanup

**Status:** deferred, not scheduled. Raised by codex against the delivered
stack (#1272 `f56ea0241`, #1291 `f647284c1`, #1292 `a2b629456`) and explicitly
out of scope there under the standing rule that bugs are fixed in a later PR.

None of these is a correctness defect. Each is a cleanup that would have widened
a stack already at its size limit.

## 1. The Wallet-level passphrase matrix duplicates focused tests

The rotation matrix at the Wallet level re-covers what the focused keyvault and
kvdb tests already prove, and does not validate anything the narrower tests
miss. Remove it, or keep at most one small public Wallet smoke per backend.
**Do not** replace it with a new integration scenario. The four generic
`ChangePassphrase` interruption tests in `controller_test.go` are in the same
category.

## 2. Signer secret reads need not widen the pass-through cache

The stack adds `GetAccountSecret` and `GetAddressSecret` to `runtimeCache`
purely as pass-throughs. If a later cleanup targets this API, have the signer
read the Store directly and drop the two cache methods rather than keeping a
cache that caches nothing.

## 3. Broader library API cleanup

Public account types still expose `db.AccountInfo` with nested `db.KeyScope` and
`db.ScopeAddrSchema`, so external callers can consume the values but cannot name
the types. Unchanged by the SQL work; worth one pass once the backends settle.

## Out of scope here

Anything that changes wallet behaviour, adds an itest case, or touches the
kvdb→SQL migration (#323), watch-only policy (#320), or account genesis (#317) —
each has its own task.
