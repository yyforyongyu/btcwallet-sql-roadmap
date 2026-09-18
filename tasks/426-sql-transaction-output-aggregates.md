# Task 426 — validate persisted transaction output aggregates

Component: **Manager Wallet publication and SQL TxStore amount validation**.

Type: **Bug fix**.

Dependencies: **None**.

## Rationale

At `sql-wallet` revision `2ae306dea8c5`, `validateCreateTxParams` checks
transaction shape and duplicate inputs but not output amounts. Maintained
publication records transactions before publishing them, so authoring checks
alone cannot prevent invalid individual or aggregate outputs from becoming
durable through SQL ingestion. `checkMempool` permits publication when the
backend lacks a mempool precheck, and `recordTxAndCredits` commits published
credits before sending the transaction. Process termination or failed
invalidation can leave that live state durable; `BuildUtxoInfo` trusts its
amounts on reopen, so new-write validation alone does not cover existing
Wallets.

## Details

- Require every decoded output and its checked aggregate to remain in
  `0..MaxSatoshi` before transaction effects are written.
- Reuse the shared direct/batch admission boundary reached by maintained
  publication and scan writes. Future migration must consume that same
  boundary; its implementation is not part of this task.
- Apply identical validation on SQLite and PostgreSQL, preserving atomic
  batch failure and useful transaction/output diagnostics.
- Before exposing a populated SQL Wallet, check its live transaction and
  credit amounts against the same individual and aggregate bounds. Refuse
  invalid live state atomically, reporting the transaction and output position
  without publishing a Wallet or making partial durable changes.
- Preserve legitimately retained terminal failed/replaced history; do not
  treat that history as live credit or silently delete it.
- Reuse existing reads and activation boundaries for this targeted check. Add
  no amount CHECK, schema migration, general audit framework, input
  normalization, amount repair, or Task 424 dependency.
- Exclude authoring arithmetic, input-source totals, fees, UTXO-row amount
  constraints, script execution, and consensus redesign.

## Suggested fix

Extend the existing shared `NewCreateTxRequest`/`validateCreateTxParams`
admission with checked output bounds before either dialect mutates state.
Reuse it for direct and batch writes rather than adding another validator
per public entry point. Reuse that predicate when checking populated live
state before Wallet publication; keep failure a read-only refusal.

## Verification

- On both dialects, direct and batch ingestion accept ordinary, zero, and
  exact-maximum output sets.
- Negative, individual-over-maximum, overflow, and aggregate-over-maximum
  values leave no raw transaction or related row; an invalid batch member
  leaves no partial batch.
- Exercise the maintained publication path to prove invalid external input
  is rejected before durable transaction effects, without a new fault seam.
- Reopen populated SQL fixtures representing interrupted publication with an
  invalid live output or aggregate. Refusal identifies the transaction/output,
  publishes no Wallet, and leaves data unchanged on both dialects. Valid live
  state and legitimately retained terminal history still open successfully.

## Definition of done

Maintained SQL ingestion rejects invalid individual and aggregate output
amounts through the shared admission boundary, with identical backend
behavior and focused rollback evidence. Populated Wallets cannot expose the
demonstrated invalid live state; targeted activation checks preserve data and
terminal history. No schema, general audit framework, graph, policy, or
authoring redesign is included.
