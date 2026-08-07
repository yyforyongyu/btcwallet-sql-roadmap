# Task 287: Detect Confirmed Conflicts On External Inputs

Status: Archived. Completed.

Archive note: SQL backends now have `tx_inputs`, insert every transaction input,
and conflict discovery merges wallet-owned UTXO spend edges with active unmined
input spenders. Verification passed:

- `go test ./wallet/internal/db/... -run 'TestCreateTxConfirmedWinnerInvalidatesExternalInputConflict|TestCreateTxConfirmReuseRepairsExternalInputConflict|TestCreateTxWithOps.*Conflict|TestHandleTxConflicts|Test.*Spendable'`
- `go test -tags itest ./wallet/internal/db/itest -run 'TestCreateTxConfirmedWinnerInvalidatesExternalInputConflict|TestCreateTxConfirmReuseRepairsExternalInputConflict'`

Source review: `.lgtm/reviews/gpt-review-tasks.md`, finding 4.

## Finding

SQL conflict discovery for a newly confirmed transaction only finds direct
conflict roots through `GetUtxoSpendByOutpoint`. That query is scoped to
wallet-owned UTXOs. If a wallet transaction spends an external input, and a
confirmed relevant transaction later spends the same external input, the
displaced unmined wallet transaction can remain active because no wallet-owned
UTXO spend edge identifies it as the conflict root.

## Rationale

Conflict detection must be based on transaction inputs, not only wallet-owned
credits. A wallet transaction can be relevant because it pays to the wallet or
spends a wallet output, but it can still share non-wallet inputs with another
transaction. Once one conflicting transaction confirms, any active unmined
wallet transaction spending the same input must be replaced or invalidated.

## Detailed Fix Plan

1. Add a durable SQL representation of transaction inputs if the current schema
   does not already provide one. The table should include at least:

   - wallet ID;
   - spending transaction ID;
   - input index;
   - previous transaction hash;
   - previous output index.

2. Add a uniqueness or lookup index for
   `(wallet_id, previous_tx_hash, previous_output_index)` over active unmined
   inputs. If status filtering cannot be expressed in the index for both
   backends, keep the status predicate in the query.

3. Populate input rows whenever `CreateTx` inserts a transaction, for both
   unmined and confirmed transactions. This must happen for every input, not
   only inputs that spend wallet-owned UTXOs.

4. On rollback, replacement, invalidation, and delete paths, ensure input rows
   follow the transaction lifecycle or remain queryable with status filtering.

5. Replace or augment `ListConflictTxns` so it discovers active unmined root
   transactions by matching the incoming confirmed transaction's inputs against
   the indexed input table.

6. Preserve current wallet-owned UTXO spend-edge handling. The input table should
   add conflict discovery coverage; it should not remove the existing spent-UTXO
   bookkeeping used for balances and rollback.

7. Revisit confirm-reuse behavior. If the confirmed transaction row already
   exists, the conflict-discovery path still needs to replay or repair missing
   replacement edges before returning.

## Tests

- Add a shared `CreateTxWithOps` test where an active unmined wallet transaction
  spends an external input, then a confirmed relevant transaction spends that
  same external input. Assert the unmined transaction is marked replaced or
  failed according to the existing conflict semantics.
- Add SQLite and PostgreSQL integration coverage for the same scenario.
- Add a confirm-reuse variant where the confirmed transaction is inserted first
  without complete replacement edges, then reprocessed and repairs the missing
  edges.
- Ensure existing wallet-owned UTXO conflict tests still pass.

## Verification

- `go test ./wallet/internal/db/...`
- `make itest-db db=sqlite`
- `make itest-db db=postgres`, when Docker/Postgres is available.
- Run `make sql` or the SQL generation/check target after schema/query changes.

## Acceptance Criteria

- Confirmed transactions detect conflicts against active unmined wallet
  transactions that share any input, including external inputs.
- Replacement history and descendant invalidation are correct for both
  wallet-owned and external-input conflict roots.
- SQLite and PostgreSQL backends implement equivalent behavior.
