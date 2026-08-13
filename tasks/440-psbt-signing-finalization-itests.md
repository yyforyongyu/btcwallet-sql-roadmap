# Task 440 — cover PSBT signing and finalization contracts

Component: **PsbtManager signing integration tests**.

Dependencies: **None**.

## Rationale

[Issue #1250][issue1250] specifically requires representative `SignPsbt` and
`FinalizePsbt` coverage. The two methods share one ownership, script, and
signature fixture, but no real-backend public test proves partial signing,
auto-signing, final script construction, or their failure boundaries across
kvdb, SQLite, and PostgreSQL.

## Details

- Exercise `SignPsbt` and `FinalizePsbt` through real kvdb, SQLite, and
  PostgreSQL Wallets.
- Register one shared public contract for all three database backends. Do not
  skip kvdb inside shared cases. Restrict backend-specific code to setup and
  teardown; require identical public assertions and stable error identities.
- Cover fully owned, partially owned, unknown, already-signed, SegWit, and
  Taproot inputs plus supported multi-party partial-signature behavior.
- Cover malformed previous-output metadata, unsupported scripts or sighashes,
  incomplete finalization, and stopped, locked, and watch-only Wallet states.
- Verify produced signatures and finalized scripts independently with the
  maintained script engine; do not establish success by reading Store rows.
- Verify errors leave the caller packet unchanged and successful signing does
  not finalize unless `FinalizePsbt` is invoked.
- Tasks 347 and 407 continue to own future authorization changes.

## Suggested fix

Add one signing-focused itest source that derives deterministic spend fixtures,
clones each packet before invocation, and independently executes successful
final scripts against their previous outputs.

## Verification

- Run the focused PsbtManager signing cases on kvdb, SQLite, and PostgreSQL.
- Verify packet immutability on every error and complete resource teardown.

## Definition of done

- Both methods have representative ownership, partial-signature, script,
  lifecycle, watch-only, and invalid-packet coverage.
- Successful final packets execute, while rejected packets remain unchanged.
- Shared cases contain no backend skip branch and add no new signing behavior.

[issue1250]: https://github.com/btcsuite/btcwallet/issues/1250
