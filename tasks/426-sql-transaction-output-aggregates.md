# Task 426 — validate persisted transaction output aggregates

Component: **SQL TxStore ingestion amount validation**.

Type: **Bug fix**.

Dependencies: **[Task 424][t424]**.

## Rationale

Transaction creation validates its own output set, but SQL TxStore also admits
transactions learned from scans, publication, migration, and batch paths. A
serialized transaction whose individual or aggregate outputs exceed the
monetary maximum must not become durable merely because it bypassed authoring.

## Details

- Add one SQL-ingestion validator for every decoded transaction output and its
  checked aggregate. Require each value and the aggregate to remain in
  `0..MaxSatoshi`.
- Invoke it before mutation in every maintained direct, batch, scan,
  publication, and migration ingestion route. Centralize it below those entry
  points so a later caller cannot bypass the check.
- Preflight every populated SQL transaction row before installing the new
  admission contract. Decode the stored bytes, recompute the txid, and refuse
  an invalid value, arithmetic overflow, or aggregate-over-maximum set with the
  exact transaction and output position.
- Keep this task merge-safe with Task 409 in either order. Amount validation
  gates the raw transaction before any normalized input row is written; input-
  graph creation neither substitutes for nor bypasses that gate.
- Apply identical validation and refusal semantics on SQLite and PostgreSQL.
  Do not repair monetary values or retain a partial batch.
- Exclude authoring, input-source totals, fees, UTXO-row amount constraints,
  transaction-input normalization, policy, script execution, and consensus
  redesign.

## Suggested fix

Add one TxStore raw-transaction admission helper that decodes and checks all
output values with bounded addition before either dialect executes its first
write. Reuse it from single and batch insertion and from a populated-data
preflight, preserving exact diagnostic identity and transaction atomicity.

## Verification

- SQLite and PostgreSQL direct, batch, scan, publication, and migration cases
  accept zero, exact-maximum, and ordinary output sets.
- Negative, individual-over-maximum, checked-overflow, and aggregate-over-
  maximum fixtures fail before any raw transaction or related row is written.
- Populated upgrades refuse each invalid class with the exact txid and output
  position and leave the prior schema and data unchanged.
- Run the same fixtures with Task 409's normalized-input migration applied
  before and after this change and prove neither merge order bypasses amount
  admission or leaves a partial input graph.

## Definition of done

- Every maintained SQL transaction-ingestion route rejects invalid individual
  and aggregate output amounts before durable mutation on both dialects.
- Populated SQL data is verified before cutover, and an invalid transaction is
  refused rather than repaired, partially imported, or hidden.
- The PR changes no creation arithmetic, input graph, UTXO constraint, fee,
  policy, script, or consensus behavior.

[t424]: 424-transaction-utxo-integrity-adr.md
