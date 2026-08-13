# Task 364 — persist ordered SQL live-watch obligations

Component: **SQL watch-generation and obligation schema**.

Type: **Gap fix**.

Dependencies: **[Task 345][t345] and [Task 400][t400]**.

## Rationale

SQL has no durable fact connecting an accepted watch-chain
script with required live registration. Producers and replay need one ordered,
wallet-scoped persistence contract before chain I/O.

## Details

- Add per-wallet watch generation and ordered obligation rows with stable
  delivery IDs, exact script/address facts, and pending/delivered state.
- Provide method-specific Store primitives that allocation and import
  transactions can call inside their own mutation. Do not add a Store callback
  or a generic notification outbox.
- Provide exact readers for generation snapshots, pending ordered replay,
  delivery lookup, and producer-side ambiguous-result reconciliation.
- Make delivery completion idempotent and wallet-scoped on both SQL dialects.
- Keep chain calls, Syncer replay, address allocation, raw import mutation, and
  public pending results out of this task.
- Add no modern-kvdb durable format.

## Suggested fix

Land one two-dialect migration and small Store API for
watch generations, obligations, and completion, with all chain I/O delegated
to later Wallet/Syncer code.

## Verification

SQLite and PostgreSQL conformance proves ordered IDs,
wallet isolation, snapshot generations, idempotent completion, restart, and
exact lookup. No test invokes a chain backend or mutates an address.

## Definition of done

SQL can persist and query ordered wallet-specific live
watch obligations durably, without producing, replaying, or exposing them.

[t345]: 345-durable-watch-registration.md
[t400]: 400-live-watch-semantics-adr.md
