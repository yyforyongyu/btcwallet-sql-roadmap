# Task 372 — coordinate multi-Wallet rollback

Component: **Manager chain dispatcher and SQL Store rollback**.

Type: **Bug fix**.

Dependencies: **[Task 334][t334] and [Task 370][t370]**.

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
- Coordinate live delivery and accepted processor work around one invocation
  of the existing database-wide rollback; leave private mechanics open.
- Include historical scan writes: a batch read from the old branch cannot
  restore stale Wallet facts after rollback. Quiesce or revalidate those
  writes using the existing Wallet processing boundary.
- Rewind affected Wallet height/hash and effects atomically through the
  existing Store operation. Wallets already below the fork stay at their own
  checkpoints and may continue background catch-up.
- Resume delivery after successful rollback. Failure, including an ambiguous
  commit, follows the terminal Manager failure policy without automatic
  reconciliation; it cannot leave a Running Manager paused forever.
- Keep manual Wallet recovery and targeted rescans wallet-scoped; they cannot
  move another Wallet's tip.
- Leave live source consumption unchanged until [Task 423][t423].
- Exclude transaction-invalidation redesign, a generic write lock, replay
  markers, a persisted Manager checkpoint, a new schema or rollback
  implementation, and cross-process fencing.

## Suggested fix

Add one disconnect operation to the prepared dispatcher around the existing
Store rollback. The private admission and quiescence mechanism is flexible if
one accepted disconnect produces one atomic rollback and one continuation
decision.

## Verification

- With two Wallet processors, prove rollback waits for accepted delivery,
  invokes the Store once, rewinds affected tips, preserves a tip below the
  fork, and then resumes in order.
- Submit a disconnected event through the prepared Manager path and prove it
  reaches only this rollback operation, never Task 370's connected-event
  fanout.
- Race rollback with delivery, Wallet registration, and stop without partial
  state or deadlock.
- Cover successful, definite-failure, and ambiguous results on SQLite and
  PostgreSQL. Errors terminate Manager without automatic readback or retry.
  Reuse the existing Store atomicity proofs.
- Pause a historical batch after reading the old branch, roll back, then
  release it; stale Wallet facts must not reappear.
- Prove manual recovery remains Wallet-scoped and the live source path remains
  unchanged.

## Definition of done

The prepared Manager path exclusively admits and applies each disconnect once
through the existing database-wide rollback after Wallet processors quiesce,
without importing the separate transaction-invalidation roadmap.

[t334]: ../archive/tasks/334-shared-chain-state.md
[t370]: ../archive/tasks/370-chain-source-delivery.md
[t423]: 423-manager-chain-source-cutover.md

[rollback-tests]: https://github.com/btcsuite/btcwallet/blob/48e7f4b7e0f3808b5022fd817d12c6a4361bb9c1/wallet/internal/db/itest/tx_store_rollback_test.go
