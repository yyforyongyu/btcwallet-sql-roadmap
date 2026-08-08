# Task 414 — bound persisted SQL UTXO amounts

Component: **SQL UTXO schema**.

Dependencies: **[Task 351][t351]**.

## Rationale

Enforce each persisted UTXO amount in
`0..MaxSatoshi` on SQLite and PostgreSQL and preflight existing negative or
over-maximum rows with an exact repair-or-refusal rule.

## Details

Implement only the behavior bounded by this task.

Exclusions: checked sums, transaction aggregates, fee arithmetic,
TxCreator validation, exact spend identity, and non-UTXO value columns. Those
creation-side arithmetic concerns remain in the Task 357 family; persisted SQL
transaction output aggregates belong to Task 426.

## Suggested fix

Add dialect-equivalent `0..MaxSatoshi` CHECK constraints in
a forward migration that first reports and deterministically refuses or
repairs each invalid existing UTXO row.

## Verification

Boundary fixtures accept zero and `MaxSatoshi`, reject the
adjacent invalid values, and prove populated migrations never install the
constraint over silently invalid data.

## Definition of done

Both dialects declaratively bound each UTXO amount and
no Go arithmetic is duplicated from the Task 357 family.

[t351]: 351-sql-utxo-integrity.md
