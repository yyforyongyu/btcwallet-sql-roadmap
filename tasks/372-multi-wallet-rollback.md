# Task 372 — coordinate multi-Wallet rollback

Component: **Manager chain dispatcher and SQL Store rollback**.

Type: **Bug fix**.

Dependencies: **[Task 334][t334], [Task 370][t370], and
[Task 371][t371]**.

## Rationale

`Store.RollbackToBlock` is database-wide, but today one Wallet Syncer can call
it while other Wallet processors continue handling the same chain. The
Manager-owned path needs one disconnect owner before it can consume the live
notification stream.

At `sql-wallet` revision `48e7f4b`, the existing
[Store rollback tests][rollback-tests] already prove atomic tip/history rewind
and rollback on failure. The missing evidence is Manager coordination around
that operation, not a second Store rollback implementation or test matrix.

## Details

- Make Task 370's dispatcher the only rollback caller in the prepared SQL
  path and admit disconnected events only through this operation. Wallet
  processors and Task 370's connected-event path must not independently apply
  or fan out the same disconnect.
- Pause new delivery and finish accepted processor work before invoking the
  existing database-wide rollback once.
- Include historical scan writes: a batch read from the old branch cannot
  restore stale Wallet facts after rollback. Quiesce or revalidate those
  writes using the existing Wallet processing boundary.
- Atomically rewind the canonical frontier and every affected Wallet tip with
  the Store's existing block and transaction detach behavior.
- Resume delivery only after the rollback result proves the requested durable
  state. An unprovable result follows the existing terminal Manager failure
  path; it cannot leave a Running Manager paused forever.
- Keep manual Wallet recovery and targeted rescans wallet-scoped; they cannot
  move the shared frontier or another Wallet's tip.
- Leave live source consumption unchanged until [Task 423][t423].
- Exclude transaction-invalidation redesign, a generic write lock, replay
  markers, schema changes without a reproduced constraint bug, and
  cross-process fencing.

## Suggested fix

Add one disconnect operation to the prepared dispatcher around the existing
Store rollback. The private admission and quiescence mechanism is flexible if
one accepted disconnect produces one atomic rollback and one continuation
decision.

## Verification

- With two Wallet processors, prove rollback waits for accepted delivery,
  invokes the Store once, rewinds all affected tips and the frontier, and then
  resumes in order.
- Submit a disconnected event through the prepared Manager path and prove it
  reaches only this rollback operation, never Task 370's connected-event
  fanout.
- Race rollback with delivery, Wallet registration, and stop without partial
  state or deadlock.
- Cover definite failure and ambiguous old, new, and indeterminate outcomes on
  SQLite and PostgreSQL; resume only from a proven applied state.
- Pause a historical batch after reading the old branch, roll back, then
  release it; stale Wallet facts must not reappear.
- Prove manual recovery remains Wallet-scoped and the live source path remains
  unchanged.

## Definition of done

The prepared Manager path exclusively admits and applies each disconnect once
through the existing database-wide rollback after Wallet processors quiesce,
without importing the separate transaction-invalidation roadmap.

[t334]: ../archive/tasks/334-shared-chain-state.md
[t370]: 370-chain-source-delivery.md
[t371]: 371-canonical-chain-frontier.md
[t423]: 423-manager-chain-source-cutover.md

[rollback-tests]: https://github.com/btcsuite/btcwallet/blob/48e7f4b7e0f3808b5022fd817d12c6a4361bb9c1/wallet/internal/db/itest/tx_store_rollback_test.go
