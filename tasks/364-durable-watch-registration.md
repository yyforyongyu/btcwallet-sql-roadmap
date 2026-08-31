# Task 364 — persist ordered SQL live-watch obligations

Component: **SQL watch-generation and obligation schema**.

Type: **Gap fix**.

Dependencies: **[Task 345][t345] and [Task 400][t400]**.

## Rationale

SQL has no durable fact connecting an accepted chain-synchronized script with
required live registration. Producers and replay need one ordered,
wallet-scoped persistence contract before chain I/O.

## Details

- Add per-wallet watch generation and ordered obligation rows with exact
  script/address facts and pending/delivered state. Leave internal row keys and
  sequence representation to the smallest conventional Store implementation.
- Provide method-specific Store primitives that allocation and import
  transactions can call inside their own mutation. Do not add a Store callback
  or a generic notification outbox.
- Provide exact readers for generation snapshots, pending ordered replay, and
  producer-side ambiguous-result reconciliation.
- Make delivery completion idempotent and wallet-scoped on both SQL dialects.
- Keep chain calls, Wallet delivery replay, address allocation, raw import
  mutation, and public delivery state out of this task.
- Add no modern-kvdb durable format.
- Keep implementation local to the existing SQL schema, Store files, and
  colocated tests. Add no public delivery identifier, parallel interface,
  arbitrary helper, production file, or standalone test file; justify the
  required migration and generated output using repository conventions.
- Document every added code block inline with why it exists and how it works;
  comments that merely restate the code do not satisfy this requirement.
- Give every unit test detailed Arrange, Act, and Assert comments explaining
  its setup, operation, and evidence. Interface mocks embed `mock.Mock`, set
  `On(...).Return(...)` expectations in Arrange with the narrowest cardinality
  (`Once()` for one required call), and call `AssertExpectations(t)` directly
  in Assert without duplicate call assertions or a `require.True` wrapper.

## Suggested fix

Land one two-dialect migration and small Store API for
watch generations, obligations, and completion, with all chain I/O delegated
to the later Wallet delivery component.

## Verification

SQLite and PostgreSQL conformance proves durable ordering, Wallet isolation,
snapshot generations, idempotent completion, restart, and exact pending reads.
No test invokes a chain backend or mutates an address.

## Definition of done

SQL can persist and query ordered wallet-specific live
watch obligations durably, without producing, replaying, or exposing them.

[t345]: 345-durable-watch-registration.md
[t400]: ../archive/tasks/400-live-watch-semantics-adr.md
