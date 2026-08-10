# Task 449 — cover multi-Wallet SQL isolation

Component: **multi-Wallet SQL integration scenario**.

Dependencies: **[Task 415][t415], [Task 429][t429], [Task 431][t431],
[Task 433][t433], [Task 434][t434], [Task 435][t435], and [Task 436][t436]**.

## Rationale

[Issue #1250][issue1250] names multi-wallet SQL isolation as a required
follow-up scenario. The maintained public Manager and Wallet contracts can be
tested before their internal chain-notification ownership changes. Running the
scenario early gives later coordination work a stable regression boundary
without making that production work part of this test-only task.

## Details

- Create two Wallets under one Manager and SQL database on the same network.
- Give each Wallet distinct accounts and addresses, fund both, mine once, and
  cross each Wallet's maintained synchronization boundary.
- Verify account, address, UTXO, transaction, and spend state never leaks
  between wallet identities while shared chain height and hash agree.
- Spend from one Wallet, confirm it, and prove the other Wallet's owned state is
  unchanged. Stop and reload both Wallets and repeat the isolation assertions.
- Run the same source on SQLite and PostgreSQL through the shared Manager
  harness.
- Exclude separate Managers, cross-process fencing, network-identity mismatch,
  stress or benchmark claims, and direct Store reads.
- Keep the PR test-only and record production defects separately.

## Suggested fix

Add one two-Wallet scenario with deterministic disjoint funding and expected
sets keyed only by public wallet identities. Reuse the public Manager, Wallet,
and synchronization harness without adding a scenario-specific coordinator.

## Verification

- Run the focused scenario on SQLite and PostgreSQL.
- Verify both Wallets load from the same Manager database after restart and all
  coordinator, Wallet, Manager, miner, and backend resources join.

## Definition of done

- Two Wallets share canonical chain facts while every wallet-owned public view
  remains isolated before and after one spend and durable reload.
- The scenario validates one Manager only and claims no cross-process fencing.
- The PR is test-only and backend-neutral.

[issue1250]: https://github.com/btcsuite/btcwallet/issues/1250
[t415]: ../archive/tasks/415-utxo-public-integration-tests.md
[t429]: 429-account-mutation-itests.md
[t431]: 431-address-allocation-lookup-itests.md
[t433]: 433-wallet-sync-test-boundary.md
[t434]: 434-transaction-creation-itests.md
[t435]: 435-transaction-publication-itests.md
[t436]: 436-transaction-read-itests.md
