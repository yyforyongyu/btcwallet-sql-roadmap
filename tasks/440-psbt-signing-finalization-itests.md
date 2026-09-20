# Task 440 — cover PSBT signing and finalization contracts

Component: **PsbtManager signing integration tests**.

Type: **Test**.

Dependencies: **[Task 407][t407]**.

## Rationale

[Issue #1250][issue1250] requires public `SignPsbt` and `FinalizePsbt` coverage.
At `sql-wallet` revision `48e7f4b`, [the handlers][source] still mutate caller
packets during work. Task 407 and its Tasks 469/347 prerequisites own final
authorization and unchanged-on-error behavior. This suite proves that contract
with real kvdb, SQLite, and PostgreSQL Wallets without repeating their focused
authorization matrices.

## Details

- Exercise `SignPsbt` and `FinalizePsbt` through real kvdb, SQLite, and
  PostgreSQL Wallets.
- Register one shared public contract for all three database backends. Do not
  skip kvdb inside shared cases. Restrict backend-specific code to setup and
  teardown; require identical public assertions and stable error identities.
- Cover fully owned, partially owned, unknown, already-signed, SegWit, and
  Taproot inputs plus supported multi-party partial-signature behavior.
- Cover malformed previous-output metadata, unsupported scripts or sighashes,
  incomplete finalization, and locked or watch-only Wallet states.
- Verify produced signatures and finalized scripts independently with the
  maintained script engine; do not establish success by reading Store rows.
- Verify errors leave the caller packet unchanged and successful signing does
  not finalize unless `FinalizePsbt` is invoked.
- Consume Task 407's final contract, including Tasks 469 and 347. Those
  production tasks retain their fixes and focused authorization regressions;
  use representative public cases here without another exhaustive matrix.

## Suggested fix

Reuse existing Signer spend fixtures, snapshot caller packets, and execute
successful final scripts independently against their previous outputs. Keep
file organization local to the existing suites rather than requiring a new
source file.

## Verification

- Run the focused PsbtManager signing cases on kvdb, SQLite, and PostgreSQL.
- Verify packet immutability on every error and complete resource teardown.

## Definition of done

- Both methods have representative ownership, partial-signature, script,
  watch-only, and invalid-packet coverage.
- Successful final packets execute, while rejected packets remain unchanged.
- Shared cases contain no backend skip branch and add no new signing behavior.

[issue1250]: https://github.com/btcsuite/btcwallet/issues/1250
[t407]: 407-psbt-finalization-authorization.md
[source]: https://github.com/btcsuite/btcwallet/blob/48e7f4b7e0f3808b5022fd817d12c6a4361bb9c1/wallet/psbt_manager.go#L1049
