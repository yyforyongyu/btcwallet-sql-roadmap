# Task 353 — retire kvdb from the modern wallet API

Component: **Manager backend selection**.

Type: **Migration**.

Dependencies: **[Task 376][t376]**.

## Rationale

After supported wallets can be converted and maintained consumers use SQL, the
modern Manager no longer needs a second kvdb backend or its public passphrase
configuration. Conversion alone is not evidence that removal is safe.

## Details

- Schedule only after Task 376 is verified, a named consumer artifact or
  release proves maintained consumers no longer select modern kvdb, and the
  retirement decision is explicit.
- Remove modern kvdb Manager selection, adapter and Vault wiring, and
  kvdb-only fields from modern configuration.
- At implementation time, remove or supersede pending roadmap clauses that
  exist solely for modern kvdb. Do not implement temporary behavior only to
  delete it.
- Keep CLI, RPC, Loader, deprecated Wallet paths, `waddrmgr`, and `wtxmgr`
  unchanged. If one still constructs the modern Manager kvdb path, the cutover
  gate is unmet.
- Preserve the legacy source format and files used as conversion inputs.

## Suggested fix

Delete the modern Manager's kvdb branch and simplify only its now-unused
configuration and Wallet-owned adapters. Record the affected task decisions in
the implementation PR rather than maintaining a speculative ledger here.

## Verification

- Prove modern Manager selection rejects kvdb and converted SQLite Wallets
  still create, start, stop, and reopen.
- Build unchanged legacy callers and audit the diff to prove no excluded
  package changed.
- Confirm no maintained consumer or remaining roadmap task requires the
  removed modern kvdb path.

## Definition of done

The modern Manager and its public configuration contain no kvdb backend after
verified conversion, completed consumer cutover, and explicit retirement,
while legacy packages and disk formats remain unchanged.

[t376]: 376-orchestrate-legacy-conversion.md
