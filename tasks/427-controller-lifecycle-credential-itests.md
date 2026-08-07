# Task 427 — cover Controller lifecycle and credential contracts

Component: **Controller lifecycle and credential integration tests**.

Dependencies: **None**.

## Rationale

[Issue #1250][issue1250] assigns Gustavo the Controller integration-test lane.
The lifecycle and credential methods share one real Wallet state machine, but
their public stopped, started, locked, unlocked, and durable passphrase
contracts are not covered together through a real SQL-backed Wallet.

## Details

- Add public integration cases for `Start`, `Stop`, `Unlock`, `Lock`, and
  `ChangePassphrase` through a real Store-backed Wallet.
- Require SQLite and register the cases once so the same source runs on
  PostgreSQL after [Task 333][t333] enables that harness.
- Cover the supported initial transition, repeated-call behavior, invalid
  transition, wrong credential, unlock timeout, explicit lock, and shutdown
  paths without asserting internal latch or goroutine shape.
- Reopen the same wallet after a successful passphrase change and prove the old
  credential is rejected and the new credential unlocks it.
- Keep the PR test-only. Do not fold in behavior from Tasks 338, 368, 369, 421,
  or another production task; record any exposed defect against its owning
  task instead.
- Exclude `Info`, `Resync`, `Rescan`, account/address APIs, RPC, deprecated
  interfaces, direct Store assertions, and production fixes.

## Suggested fix

Add one focused Controller lifecycle itest source with a shared real-wallet
fixture and bounded transition helpers. Drive every assertion through public
Controller and Wallet calls, and recreate Manager, Wallet, and Controller
objects for the durable credential check.

## Verification

- Run the focused SQLite Controller lifecycle case selection.
- After Task 333 merges, run the same registered cases on PostgreSQL without a
  source fork.
- Verify every timeout is bounded and each success and failure path stops all
  wallet, chain, miner, and backend resources.
- Verify the PR changes only integration-test or narrowly reusable harness
  files.

## Definition of done

- The five lifecycle and credential methods have representative public success,
  stable-error, state-transition, and timeout coverage on a real SQLite Wallet.
- A changed passphrase remains authoritative after a full durable reopen.
- The cases remain backend-neutral for PostgreSQL enablement.
- The PR is test-only; exposed production defects remain separate tasks.

[issue1250]: https://github.com/btcsuite/btcwallet/issues/1250
[t333]: 333-postgres-manager-harness.md
