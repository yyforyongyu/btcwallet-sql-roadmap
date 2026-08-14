# Task 450 — compare public wallet state with bitcoind

Component: **bitcoind compatibility integration scenario**.

Type: **Test**.

Dependencies: **[Task 415][t415], [Task 431][t431], [Task 433][t433],
[Task 434][t434], [Task 435][t435], and [Task 436][t436]**.

## Rationale

The [Issue #1250 compatibility proposal][compat] calls for a deterministic
integration/property test that drives btcwallet and bitcoind through the same
regtest history and compares only compatible public behavior and chain-derived
facts. No current task owns that scenario.

## Details

- Register the same btcwallet flow for kvdb, SQLite, and PostgreSQL with
  bitcoind as the chain backend, then generate public receiving addresses. Do
  not skip kvdb inside the shared flow.
- Fund those addresses from bitcoind, mine blocks, cross Task 433's exact sync
  boundary, and compare height, hash, watched UTXOs, UTXO-sum balance,
  transaction history, confirmations, and mined or unmined state.
- Create and broadcast one btcwallet spend, compare mempool-visible facts, mine
  it, synchronize, and repeat the chain-derived comparisons.
- Compare only facts both systems model equivalently. Do not compare account,
  change, label, or implementation-specific wallet metadata.
- Keep the initial scenario deterministic. Exclude seeded randomization,
  fuzzing, reorgs, multi-Wallet behavior, performance claims, and direct Store
  inspection.

## Suggested fix

Add one bitcoind-backed scenario with a small comparison helper that normalizes
outpoints, amounts, transaction IDs, confirmations, height, and hash from each
system's public view. Keep expected ownership in the test fixture.

## Verification

- Run the focused scenario with `chain=bitcoind` on kvdb, SQLite, and
  PostgreSQL.
- Verify comparison failures identify the exact normalized fact and all
  bitcoind, Wallet, Manager, miner, and backend resources stop.

## Definition of done

- btcwallet and bitcoind agree on the deterministic receive and spend history's
  compatible chain-derived facts before and after confirmation.
- The scenario makes no equivalence claim about wallet-specific metadata.
- The shared scenario contains no backend skip branch and remains deterministic
  and not a fuzz or benchmark target.

[compat]: https://github.com/btcsuite/btcwallet/issues/1250#issuecomment-4563982303
[t415]: ../archive/tasks/415-utxo-public-integration-tests.md
[t431]: 431-address-allocation-lookup-itests.md
[t433]: ../archive/tasks/433-wallet-sync-test-boundary.md
[t434]: 434-transaction-creation-itests.md
[t435]: 435-transaction-publication-itests.md
[t436]: 436-transaction-read-itests.md
