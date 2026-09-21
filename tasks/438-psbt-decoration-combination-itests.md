# Task 438 — cover PSBT decoration and combination contracts

Component: **PsbtManager packet integration tests**.

Type: **Test**.

Dependencies: **None**.

## Rationale

[Issue #1250][issue1250] requires real-wallet integration coverage for every
maintained public `wallet/` API. `DecorateInputs` and `CombinePsbt` are the
PsbtManager packet transformations, but their real-wallet ownership and error
boundaries lack focused public coverage. At `sql-wallet` revision `48e7f4b`,
[TestDecorateInputsPartialError][source] deliberately preserves in-place
enrichment before a later input fails. `CombinePsbt` instead constructs a
separate result. Neither needs another unit-level field-merge matrix.

## Details

- Exercise `DecorateInputs` and `CombinePsbt` only through a real Wallet.
- Register the same public contract for kvdb, SQLite, and PostgreSQL. Do not
  skip kvdb inside shared cases. Restrict backend-specific code to setup and
  teardown; require identical public assertions and stable error identities.
- Cover owned SegWit and Taproot inputs, mixed owned and unknown inputs, both
  `skipUnknown` modes, and malformed packets.
- Combine compatible copies with disjoint signatures or metadata. Cover
  different unsigned transactions, empty input, and a representative field
  conflict; reuse existing unit tests for detailed merge permutations.
- Preserve unrelated metadata. Decoration enriches the supplied packet in
  place and may retain earlier enrichment after a later input fails; do not
  add rollback semantics. Combination returns a separate result and leaves
  source packets unchanged on success and rejection.
- Fix only defects revealed by the required public cases and retain their
  regressions in this task.
- Do not define future metadata authority or signing rules owned by Tasks 346,
  347, and 407.

## Suggested fix

Reuse deterministic wallet and foreign input fixtures for representative
public calls. Compare source and result packets according to each method's
ownership contract, without Store reads or a mandatory new source file.

## Verification

- Run the focused PsbtManager packet cases on kvdb, SQLite, and PostgreSQL.
- Verify combination preserves its sources and decoration retains its
  documented in-place behavior; all Wallet, Manager, miner, and backend
  resources stop.

## Definition of done

- Both packet transformations have representative success, mixed-ownership,
  malformed, conflict, and caller-ownership coverage.
- Shared cases use public APIs on real kvdb, SQLite, and PostgreSQL Wallets
  without a backend skip branch.

[issue1250]: https://github.com/btcsuite/btcwallet/issues/1250
[source]: https://github.com/btcsuite/btcwallet/blob/48e7f4b7e0f3808b5022fd817d12c6a4361bb9c1/wallet/psbt_manager_test.go#L41
