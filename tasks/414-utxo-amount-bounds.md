# Task 414 — bound persisted SQL UTXO amounts

Component: **SQL UTXO amount integrity backstop**.

Type: **Refactor**.

Dependencies: **[Task 424][t424]**.

## Rationale

The current UTXO schema rejects negative amounts but has no MaxSatoshi upper
bound, and public UTXO conversion trusts the stored amount. This task adds a
defensive persistence backstop; the missing constraint alone does not prove
that a maintained public API currently accepts an invalid amount.

## Details

- Preserve the accepted per-UTXO range 0..MaxSatoshi on SQLite and PostgreSQL.
  Implement only the additional persistence protection justified by Task 424.
- Keep current valid public behavior unchanged. Identify existing protection
  before adding duplicate validation; record any distinct reachable public
  arithmetic defect under its actual owner.
- If a schema change is necessary, validate existing data before cutover and
  refuse out-of-range rows without clamping, deleting, or inventing amounts.
- The amount boundary is independent of Task 351's tenancy representation and
  Task 409's input graph; neither is a hard prerequisite.
- Exclude checked sums, output aggregates, fee arithmetic, TxCreator changes,
  exact spend identity, and other value columns. Task 426 owns transaction
  output-set admission.

## Suggested fix

A narrow upper-bound constraint in a forward migration is one likely
backstop. Reuse existing validation and avoid changing row ownership or adding
an amount-validation framework.

## Verification

- On both dialects, accept zero and MaxSatoshi and reject adjacent invalid
  values at the accepted persistence boundary.
- Preserve valid populated data and refuse invalid existing rows atomically.
- Confirm maintained public UTXO results remain unchanged for valid amounts.

## Definition of done

The justified UTXO amount backstop is complete on both dialects without
duplicating transaction arithmetic or becoming a prerequisite for conversion.

[t424]: 424-transaction-utxo-integrity-adr.md
