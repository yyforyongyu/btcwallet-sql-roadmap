# Task 364 — persist ordered SQL live-watch obligations

Superseded on 2026-09-10: the accepted fail-stop and startup-reconstruction
decision eliminates separate durable watch obligations and completion flags.
Existing address/script records identify required watches, not live readiness.
No implementation PR was found. [Task 402][t402] retains its
existing live-registration outcome without this persistence prerequisite.

Component: **SQL live-watch persistence**.

Type: **Gap fix**.

Dependencies: **[Task 345][t345] and [Task 400][t400]**.

## Rationale

SQL can commit an address or script before the chain backend accepts its live
watch. A crash or transient chain failure in that interval leaves durable
wallet state with no durable record that registration is still required. An
address returned through lnd could then receive funds the running wallet does
not observe.

Producers and startup replay need one wallet-scoped ordering fact that survives
restart without exposing delivery bookkeeping publicly.

## Details

- Persist wallet-scoped live-watch obligations with enough script/address and
  ordering information to deliver them deterministically.
- Let an address or import mutation record its obligation in the same SQL
  transaction as the owned wallet fact.
- Support coherent startup snapshots, ordered pending reads, idempotent
  completion, and exact durable-state inspection after an ambiguous result.
- Keep chain I/O outside Store transactions and leave delivery and retry to
  Task 402.
- Support SQLite and PostgreSQL without adding a modern-kvdb format.
- Do not expose a delivery identifier, generic outbox, Store callback, public
  replay API, or chain-specific state.

## Suggested fix

Add a conventional two-dialect persistence shape and narrow Store operations
for recording, reading, and completing obligations. Table layout, sequence
representation, and query decomposition are not part of the task contract.

## Verification

- On both SQL dialects, record obligations with their producing mutation and
  verify rollback removes both.
- Verify wallet isolation, durable order, coherent snapshots, idempotent
  completion, and reopen of pending work.
- Exercise ambiguous-result inspection without performing chain calls.

## Definition of done

SQL producers can atomically retain an ordered live-watch obligation that
survives restart and can be completed idempotently, without delivering it or
exposing new public state.

[t345]: ../../tasks/345-durable-watch-registration.md
[t400]: 400-live-watch-semantics-adr.md
[t402]: ../../tasks/402-replay-live-watch-obligations.md
