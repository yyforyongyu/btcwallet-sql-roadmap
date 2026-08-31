# Task 345 — persist SQL account NoChainSync state

Component: **SQL account schema and AccountStore persistence**.

Type: **Gap fix**.

Dependencies: **[Task 340][t340] and [Task 387][t387]**.

## Rationale

SQL cannot record that an account must stay out of automatic chain
synchronization. That behavior is independent from whether the wallet holds
private material and must survive reopen before public callers can select it.

## Details

- Add one immutable `accounts.no_chain_sync BOOLEAN NOT NULL DEFAULT FALSE`
  column in SQLite and PostgreSQL. Backfill every existing account as false
  before enforcing the non-null constraint in each dialect.
- Include `NoChainSync` in account reads and method-specific creation inputs.
  Update every maintained SQL derived and imported caller in the same PR to
  create ordinary accounts with false.
- Reject any attempt to change the value after account creation. Do not add a
  discovery-mode enum, account kind, custody column, key-only marker, or new
  table.
- Treat accountless raw imports as implicitly chain-synchronized; do not
  invent an account row or a no-chain-sync raw-import form.
- Keep wallet-level signing custody and `IsWatchOnly` unchanged. This property
  says only whether automatic chain synchronization includes the account.
- Keep public account request and result shapes in Task 392 and consumer
  policy in Tasks 393, 399, and 452.
- Keep implementation local to the existing account schema, queries, Store,
  and colocated tests. The required migration and regenerated sqlc files are
  the only expected new or generated files; add no arbitrary helper,
  production file, or standalone test file.
- Document every added code block inline with why it exists and how it works;
  comments that merely restate the code do not satisfy this requirement.
- Give every unit test detailed Arrange, Act, and Assert comments explaining
  its setup, operation, and evidence. Interface mocks embed `mock.Mock`, set
  `On(...).Return(...)` expectations in Arrange with the narrowest cardinality
  (`Once()` for one required call), and call `AssertExpectations(t)` directly
  in Assert without duplicate call assertions or a `require.True` wrapper.

## Suggested fix

Land one two-dialect account migration, regenerated queries, and the smallest
Store conversion needed to carry the immutable boolean through existing
account rows and creation calls.

## Verification

Migration fixtures prove old accounts reopen with `NoChainSync=false`, new
rows preserve both boolean values, and updates cannot change the value on
SQLite and PostgreSQL. Fresh and migrated schemas reject an explicit NULL on
insert or update instead of silently treating it as false. Maintained creation
callers explicitly exercise false, Store conversion rejects malformed
persisted values, and generated output matches source SQL.

## Definition of done

Every SQL account has one immutable, non-null `NoChainSync` value after
migration, all maintained SQL creation callers supply it, and no public
selection behavior, signing-custody change, chain call, enum, account kind, or
new table is added.

[t340]: 340-durable-database-identity.md
[t387]: 387-account-creation-identity-adr.md
