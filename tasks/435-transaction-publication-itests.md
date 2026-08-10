# Task 435 — cover transaction publication contracts

Component: **TxPublisher integration tests**.

Dependencies: **None**.

## Rationale

[Issue #1250][issue1250] assigns Abdullahi the TxPublisher integration-test
lane. `CheckMempoolAcceptance` and `Broadcast` share one network-publication
boundary, but their public success, backend rejection, and Wallet lifecycle
contracts lack focused coverage against a deterministic chain backend.

## Details

- Exercise `CheckMempoolAcceptance` and `Broadcast` through a real SQL-backed
  Wallet and deterministic regtest backend.
- Require SQLite and register one backend-neutral source for PostgreSQL after
  [Task 333][t333].
- Cover an accepted spend, a representative policy rejection, malformed or
  conflicting transactions, already-known behavior, and stopped Wallet state.
- Prove acceptance checks do not publish and successful broadcast becomes
  observable through maintained public wallet APIs before and after mining.
- Keep current public publication semantics. Do not anticipate invalidation,
  graph admission, or crash-settlement behavior owned by Task 412.
- Exclude transaction creation, reads, labels, direct Store assertions, RPC,
  deprecated helpers, and production fixes.

## Suggested fix

Add one TxPublisher itest source that creates deterministic accepted and
rejected transactions, observes the chain backend with bounded event-specific
waits, and asserts Wallet results only through maintained public APIs.

## Verification

- Run the focused SQLite TxPublisher case selection against each supported
  chain backend.
- After Task 333 merges, run the same registered cases on PostgreSQL.
- Verify acceptance-only cases leave the mempool unchanged and all failure
  paths release Wallet, Manager, miner, and backend resources.
- Verify the PR remains test-only.

## Definition of done

- Both TxPublisher methods have representative acceptance, publication,
  rejection, lifecycle, and already-known integration coverage.
- Successful broadcast is confirmed through public behavior, while an
  acceptance check alone never publishes.
- The cases use a real SQLite Wallet and remain backend-neutral.
- Any exposed production defect is recorded separately.

[issue1250]: https://github.com/btcsuite/btcwallet/issues/1250
[t333]: ../archive/tasks/333-postgres-manager-harness.md
