# Task 289: Carry UTXO Spendability From Store

Status: Archived. Completed.

Archive note: `db.UtxoInfo` now carries store-derived spendability,
`Wallet.buildWalletUtxoFromStore` uses `info.Spendable` with only coinbase
maturity layered on top, and automatic coin selection excludes non-spendable
UTXOs. Verification passed:

- `go test ./wallet -run 'TestManagerCreateRejectsLoadedWallet|TestManagerCreateRejectsStartedWallet|TestManagerCreateReleasesGuardOnFailure|TestResolveScanTargets|TestStoreScanHorizonsTargeted|TestGetUtxo|TestFilterEligibleOutputsExcludesNonSpendable|TestImportPrivateKeyKVDBSucceeds'`
- `go test ./wallet/internal/db/... -run 'TestCreateTxConfirmedWinnerInvalidatesExternalInputConflict|TestCreateTxConfirmReuseRepairsExternalInputConflict|TestCreateTxWithOps.*Conflict|TestHandleTxConflicts|Test.*Spendable'`

Source review: `.lgtm/reviews/gpt-review-tasks.md`, finding 6.

## Finding

`Wallet.buildWalletUtxoFromStore` derives public `Utxo.Spendable` from the
wallet-level watch-only flag. That means imported public-key or script-only
UTXOs can be reported and selected as spendable in an otherwise spendable
wallet, even when the wallet has no private key material capable of signing that
specific output.

Current tests also encode this policy by asserting imported UTXOs are spendable
whenever the wallet itself is not watch-only.

## Rationale

Wallet-level watch-only state is not enough to decide whether a particular UTXO
can be spent automatically. Derived wallet outputs are spendable for a spendable
wallet, but imported public keys, watch-only scripts, and script-only imports
need per-address or per-output capability.

Automatic coin selection should not select outputs that the signer cannot sign.
The public `Utxo.Spendable` field should reflect the actual ability to spend the
output, after coinbase maturity and lock state are considered.

## Detailed Fix Plan

1. Add a spendability field to the Store contract. The least invasive shape is a
   boolean on `db.UtxoInfo`, for example `Spendable` or `CanSign`, populated by
   every backend.

2. Define the backend-neutral semantics clearly:

   - false for all outputs when the wallet is watch-only;
   - true for derived outputs in a spendable wallet;
   - true for imported private-key outputs when the private key is available;
   - false for imported public-key-only outputs;
   - false for script-only outputs unless the wallet has all required signing
     material and the signer path can actually satisfy the script.

3. Update SQL address/UTXO queries to join enough address-secret or account data
   to populate the spendability field. Avoid deriving spendability from account
   number alone; imported address rows need their own capability.

4. Update the kvdb UTXO adapter to preserve the watch-only/private-key
   capability it can already infer from managed address metadata instead of
   dropping it during UTXO enrichment.

5. Update `buildWalletUtxoFromStore` to start from `info.Spendable` and then
   apply coinbase maturity. Locked outputs should continue to be reported as
   locked, and automatic selection should keep excluding locked outputs through
   the existing filter.

6. Update coin selection/filtering code so automatic funding paths exclude
   non-spendable UTXOs before signing. Manual listing APIs can still return them
   with `Spendable=false`.

7. Revisit tests that currently assert imported watch-only outputs survive
   filtering. Replace them with narrower tests that distinguish imported private
   keys from imported public/script-only outputs.

## Tests

- Add `GetUtxo` and `ListUnspent` tests for imported private-key outputs in a
  spendable wallet: `Spendable=true`.
- Add tests for imported public-key-only and script-only outputs in a spendable
  wallet: `Spendable=false`.
- Add coin-selection tests proving automatic funding excludes non-spendable
  imported outputs.
- Add backend adapter tests for kvdb, SQLite, and PostgreSQL to verify the store
  populates the new field consistently.

## Verification

- `go test ./wallet -run 'TestGetUtxo|TestListUnspent|TestFilterEligibleOutputs'`
- `go test ./wallet/internal/db/...`
- `make itest-db db=sqlite`
- `make itest-db db=postgres`, when Docker/Postgres is available.
- Run SQL generation/checks if query or model changes are needed.

## Acceptance Criteria

- `Utxo.Spendable` is based on per-output signing capability, not only
  wallet-level watch-only state.
- Automatic coin selection does not select public/script-only imported outputs
  that the wallet cannot sign.
- All Store backends expose equivalent spendability semantics.
