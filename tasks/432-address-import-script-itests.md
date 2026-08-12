# Task 432 — cover address import and script contracts

Component: **AddressManager import and script integration tests**.

Dependencies: **None**.

## Rationale

[Issue #1250][issue1250] assigns Gustavo the AddressManager integration-test
lane. `ImportPublicKey`, `ImportTaprootScript`, `ScriptForOutput`, and
`GetDerivationInfo` share imported ownership and script-resolution state, but
their public errors and durable reload behavior lack focused real-backend
coverage across kvdb, SQLite, and PostgreSQL.

## Details

- Exercise the four public import/script methods through a real Store-backed
  Wallet, using deterministic public keys, Taproot scripts, and outputs.
- Register one shared public contract for kvdb, SQLite, and PostgreSQL. Do not
  skip kvdb inside shared cases. Restrict backend-specific code to setup and
  teardown; require identical public assertions and stable error identities.
- Cover valid imports, repeated identical imports, malformed or unsupported
  inputs, foreign outputs, stopped or locked states where defined, script
  lookup, and derivation information for derived and imported ownership forms.
- Reopen after successful imports and prove public script resolution and
  derivation information remain identical without reimport.
- Keep watch-only policy, Vault ownership, raw-import persistence, and live-watch
  delivery in their accepted production tasks.
- Exclude allocation/list APIs, private-key extraction, direct Store reads,
  RPC, deprecated interfaces, and unrelated production changes.

## Suggested fix

Add one AddressManager import/script itest source with deterministic key and
Taproot fixtures. Derive expected scripts independently in test code, assert
ownership only through public APIs, and recreate Manager and Wallet objects for
reload checks.

## Verification

- Run the focused AddressManager import/script cases on kvdb, SQLite, and
  PostgreSQL.
- Verify duplicate and invalid imports leave no partial public ownership and
  every case shuts down wallet, chain, miner, and backend resources.

## Definition of done

- All four import/script APIs have representative success, duplicate, invalid,
  foreign-output, lifecycle, and durable-reload coverage.
- Public script and derivation results survive a fresh reopen unchanged.
- Tests use real kvdb, SQLite, and PostgreSQL storage, with no backend skip
  branch in the shared contract.
- No new policy, Vault, or watch-delivery behavior is introduced.

[issue1250]: https://github.com/btcsuite/btcwallet/issues/1250
