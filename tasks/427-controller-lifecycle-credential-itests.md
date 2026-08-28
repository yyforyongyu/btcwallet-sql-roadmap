# Task 427 — cover remaining Controller credential contracts

Component: **Controller credential integration tests**.

Type: **Test**.

Dependencies: **[Task 331][t331]**.

## Rationale

[Issue #1250][issue1250] assigns Gustavo the Controller integration-test lane.
Merged [PR #1300][pr1300] covers basic `Unlock`, `Lock`, and `Info`;
[Task 368][t368] removes the individual lifecycle methods and their obsolete
cases. The remaining credential slice is passphrase change, timed unlock, and
their durable behavior through a real persistent Wallet.

## Details

- Add public integration cases for `ChangePassphrase` and the uncovered timed
  `Unlock` behavior through a real Store-backed Wallet.
- Register the same public contract for kvdb, SQLite, and PostgreSQL. Do not
  skip kvdb inside shared cases. Restrict backend-specific code to setup and
  teardown; require identical public assertions and stable error identities.
- Cover successful passphrase change, wrong current credentials, invalid new
  credentials, locked and unlocked states where defined, and an
  unlock timeout that returns the Wallet to its public locked state.
- Stop the Manager after a successful passphrase change, construct a fresh
  Manager over the same database, select the Wallet from its Start result, and
  prove the old credential is rejected and the new credential unlocks it.
- Do not absorb behavior owned by Tasks 338, 369, 421, or another task.
- Exclude the Controller methods already covered by PR #1300, `Info`, `Resync`,
  `Rescan`, account/address APIs, RPC, deprecated interfaces, direct Store
  assertions, lifecycle behavior owned by Task 368, and unrelated production
  changes.

## Suggested fix

Extend the focused Controller itest source with a real-wallet credential case
and bounded timeout observation. Drive every assertion through public
Controller and Wallet calls, and recreate Manager, Wallet, and Controller
objects for the durable credential check.

## Verification

- Run the focused Controller credential cases on kvdb, SQLite, and PostgreSQL
  without a source fork.
- Verify every timeout is bounded and each success and failure path stops all
  wallet, chain, miner, and backend resources.

## Definition of done

- `ChangePassphrase` and timed `Unlock` have representative public success,
  stable-error, state-transition, and timeout coverage on real kvdb, SQLite,
  and PostgreSQL Wallets.
- A changed passphrase remains authoritative after a full durable reopen.
- Shared cases contain no backend skip branch.

[issue1250]: https://github.com/btcsuite/btcwallet/issues/1250
[pr1300]: https://github.com/btcsuite/btcwallet/pull/1300
[t331]: 331-manager-wallet-lifecycle.md
[t368]: 368-wallet-startup-shutdown.md
