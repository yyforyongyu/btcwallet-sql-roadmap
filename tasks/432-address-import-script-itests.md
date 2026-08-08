# Task 432 — cover address import and script contracts

Component: **AddressManager import and script integration tests**.

Dependencies: **None**.

## Rationale

[Issue #1250][issue1250] assigns Gustavo the AddressManager integration-test
lane. `ImportPublicKey`, `ImportTaprootScript`, `ScriptForOutput`, and
`GetDerivationInfo` share imported ownership and script-resolution state, but
their public errors and durable reload behavior lack focused real-SQL coverage.

## Details

- Exercise the four public import/script methods through a real Store-backed
  Wallet, using deterministic public keys, Taproot scripts, and outputs.
- Require SQLite and register one backend-neutral source for PostgreSQL after
  [Task 333][t333].
- Cover valid imports, repeated identical imports, malformed or unsupported
  inputs, foreign outputs, stopped or locked states where defined, script
  lookup, and derivation information for derived and imported ownership forms.
- Reopen after successful imports and prove public script resolution and
  derivation information remain identical without reimport.
- Keep watch-only policy, Vault ownership, raw-import persistence, and live-watch
  delivery in their accepted production tasks. An exposed defect becomes a
  separate task rather than a test-only fix.
- Exclude allocation/list APIs, private-key extraction, direct Store reads,
  RPC, deprecated interfaces, and production behavior changes.

## Suggested fix

Add one AddressManager import/script itest source with deterministic key and
Taproot fixtures. Derive expected scripts independently in test code, assert
ownership only through public APIs, and recreate Manager and Wallet objects for
reload checks.

## Verification

- Run the focused SQLite AddressManager import/script case selection.
- After Task 333 merges, run the identical registration on PostgreSQL.
- Verify duplicate and invalid imports leave no partial public ownership and
  every case shuts down wallet, chain, miner, and backend resources.
- Verify the PR remains limited to integration-test and harness files.

## Definition of done

- All four import/script APIs have representative success, duplicate, invalid,
  foreign-output, lifecycle, and durable-reload coverage.
- Public script and derivation results survive a fresh reopen unchanged.
- Tests use real SQLite storage and remain backend-neutral for PostgreSQL.
- No policy, Vault, watch delivery, or other production behavior is changed.

[issue1250]: https://github.com/btcsuite/btcwallet/issues/1250
[t333]: 333-postgres-manager-harness.md
