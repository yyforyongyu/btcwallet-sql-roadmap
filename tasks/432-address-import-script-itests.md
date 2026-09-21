# Task 432 — cover address import and script contracts

Component: **AddressManager import and script integration tests**.

Type: **Test**.

Dependencies: **None**.

## Rationale

[Issue #1250][issue1250] assigns Gustavo the AddressManager integration-test
lane. `ImportPublicKey`, `ImportTaprootScript`, `ScriptForOutput`, and
`GetDerivationInfo` share imported ownership and script-resolution state, but
their public errors and durable reload behavior lack focused real-backend
coverage across kvdb, SQLite, and PostgreSQL. At `sql-wallet` revision
`48e7f4b`, [derivationForAddressInfo][source] explicitly rejects a raw import's
missing derivation path; script ownership does not create a BIP32 identity.

## Details

- Exercise public-key and Taproot-script imports through a real Wallet, using
  deterministic keys, scripts, and outputs. Resolve their scripts publicly;
  `GetDerivationInfo` for raw imports matches `ErrDerivationPathNotFound`.
- Use one derived-address control to verify its public derivation result,
  without repeating allocation or lookup matrices.
- Register one shared public contract for kvdb, SQLite, and PostgreSQL. Do not
  skip kvdb inside shared cases. Restrict backend-specific code to setup and
  teardown; require identical public assertions and stable error identities.
- Cover valid and repeated imports, malformed or unsupported inputs, foreign
  outputs, and applicable locked-state behavior. Apply each case only to the
  ownership form that can represent it; retain the accepted custody policy.
- Reopen after successful imports and prove public script resolution and
  derivation information remain identical without reimport.
- Keep watch-only policy, Vault ownership, raw-import persistence, and live-watch
  delivery in their accepted production tasks.
- Exclude allocation/list APIs, private-key extraction, direct Store reads,
  RPC, deprecated interfaces, and unrelated production changes.

## Suggested fix

Reuse suitable AddressManager fixtures for the two import forms and derived
control. Derive expected scripts independently and compare public results
across a fresh Manager startup without prescribing a new test file.

## Verification

- Run the focused AddressManager import/script cases on kvdb, SQLite, and
  PostgreSQL.
- Verify duplicate and invalid imports leave no partial public ownership and
  every case shuts down wallet, chain, miner, and backend resources.

## Definition of done

- All four import/script APIs have representative success, duplicate, invalid,
  foreign-output, and durable-reload coverage.
- Public script and derivation results survive a fresh reopen unchanged.
- Tests use real kvdb, SQLite, and PostgreSQL storage, with no backend skip
  branch in the shared contract.
- No new policy, Vault, or watch-delivery behavior is introduced.

[issue1250]: https://github.com/btcsuite/btcwallet/issues/1250
[source]: https://github.com/btcsuite/btcwallet/blob/48e7f4b7e0f3808b5022fd817d12c6a4361bb9c1/wallet/address_manager.go#L1495
