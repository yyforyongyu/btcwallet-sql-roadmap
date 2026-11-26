# Task 449 — cover multi-Wallet SQL isolation

Component: **multi-Wallet SQL integration scenario**.

Type: **Test**.

Dependencies: **[Task 331][t331], [Task 415][t415], [Task 423][t423],
[Task 429][t429], [Task 433][t433], [Task 434][t434], [Task 435][t435],
[Task 436][t436], and [Task 455][t455]**.

## Rationale

[Issue #1250][issue1250] requires multi-Wallet SQL isolation. The
[existing two-Wallet scenario][source] proves transfer and coin isolation.
Extend it to cover public account and address views and durable isolation
after a fresh Manager reopens the same database. [Task 423][t423] retains
the required live delivery boundary.

## Details

- Use the existing SQL scenario with two Wallets under one Manager. Give
  each Wallet distinct accounts and addresses and preserve the transfer and
  coin assertions across the maintained synchronization boundary.
- Verify each Wallet reports only its own accounts and addresses while both
  see the same canonical chain state.
- Stop the Manager, open a fresh Manager over the same database, and check
  that account, address, UTXO, transaction and spend state remain isolated
  in the Wallets returned by Start.
- [Task 472][t472] owns the concurrent historical-rescan/live-delivery
  scenario split out on 2026-09-19. This task retains baseline receipt,
  spend, and reopen isolation; neither test task depends on the other.
- Run the same source on SQLite and PostgreSQL through the shared Manager
  harness.
- Do not register kvdb: its Manager intentionally supports one Wallet per
  database, so this SQL-only multi-Wallet contract has no kvdb equivalent. The
  common single-Wallet scenarios own kvdb coverage.
- Exclude separate Managers, cross-process fencing, network-identity mismatch,
  historical rescans, stress or benchmark claims, and direct Store reads.

## Suggested fix

Extend the existing two-Wallet scenario and key expected results by
public Wallet identity. Reuse its funding, coin assertions and
synchronization harness. Task 431's separate address coverage does not
gate this scenario.

## Verification

- Run the focused scenario on SQLite and PostgreSQL.
- Verify the test registration documents the intentional kvdb exclusion
  instead of reaching it through a runtime skip branch.
- Verify both Wallets start from the same Manager database after fresh Manager
  construction; all Wallet, Manager, miner, and backend resources join.

## Definition of done

- Two Wallets share canonical chain facts while every wallet-owned public view
  remains isolated through receipt, one spend, live chain movement, and
  durable reopen.
- The scenario validates one Manager only and claims no cross-process fencing.
- The scenario is SQL-backend-neutral; kvdb remains explicitly out of scope
  because it cannot represent the scenario.

[issue1250]: https://github.com/btcsuite/btcwallet/issues/1250
[t472]: 472-multi-wallet-rescan-isolation-scenario.md
[t331]: ../archive/tasks/331-manager-wallet-lifecycle.md
[t415]: ../archive/tasks/415-utxo-public-integration-tests.md
[t429]: ../archive/tasks/429-account-mutation-itests.md
[t431]: ../archive/tasks/431-address-allocation-lookup-itests.md
[t433]: ../archive/tasks/433-wallet-sync-test-boundary.md
[t434]: ../archive/tasks/434-transaction-creation-itests.md
[t435]: ../archive/tasks/435-transaction-publication-itests.md
[t436]: ../archive/tasks/436-transaction-read-itests.md
[t455]: ../archive/tasks/455-exact-wallet-sync-status.md
[t423]: ../archive/tasks/423-manager-chain-source-cutover.md
[source]: https://github.com/btcsuite/btcwallet/blob/70f5629b6f5a75b29582eac9251c804d61bd7b20/itest/manager_test.go#L737
