# Task 435 — cover transaction publication contracts

Component: **TxPublisher integration tests**.

Type: **Test**.

Dependencies: **None**.

## Rationale

[Issue #1250][issue1250] assigns Abdullahi the TxPublisher integration-test
lane. `CheckMempoolAcceptance` and `Broadcast` share one network-publication
boundary, but their public success, backend rejection, and Wallet lifecycle
contracts lack focused coverage against a deterministic chain backend.

## Details

- Exercise `CheckMempoolAcceptance` and `Broadcast` through real kvdb, SQLite,
  and PostgreSQL Wallets and a deterministic regtest backend.
- Register one shared public contract for all three database backends. Do not
  skip kvdb inside shared cases. Restrict backend-specific code to setup and
  teardown; require identical public assertions and stable error identities.
- Cover an accepted spend, a representative policy rejection, malformed or
  conflicting transactions, already-known behavior, and stopped Wallet state.
- Prove acceptance checks do not publish and successful broadcast becomes
  observable through maintained public wallet APIs before and after mining.
- Keep current public publication semantics. Do not anticipate invalidation,
  graph admission, or crash-settlement behavior owned by Task 412.
- Exclude transaction creation, reads, labels, direct Store assertions, RPC,
  deprecated helpers, and unrelated production changes.

## Suggested fix

Add one TxPublisher itest source that creates deterministic accepted and
rejected transactions, observes the chain backend with bounded event-specific
waits, and asserts Wallet results only through maintained public APIs.

## Verification

- Run the focused TxPublisher cases on kvdb, SQLite, and PostgreSQL against
  each supported chain backend.
- Verify acceptance-only cases leave the mempool unchanged and all failure
  paths release Wallet, Manager, miner, and backend resources.

## Definition of done

- Both TxPublisher methods have representative acceptance, publication,
  rejection, lifecycle, and already-known integration coverage.
- Successful broadcast is confirmed through public behavior, while an
  acceptance check alone never publishes.
- Shared cases use real kvdb, SQLite, and PostgreSQL Wallets without a backend
  skip branch.

[issue1250]: https://github.com/btcsuite/btcwallet/issues/1250
