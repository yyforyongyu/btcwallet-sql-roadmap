# Task 345 — persist SQL account NoChainSync state

Component: **SQL account schema and AccountStore persistence**.

Type: **Gap fix**.

Dependencies: **[Task 340][t340] and [Task 387][t387]**.

## Rationale

SQL account rows cannot record that an account must stay out of automatic
chain synchronization. Lnd's purpose-1017 key-family accounts hold protocol
keys rather than receiving addresses, so treating them as ordinary accounts
adds irrelevant recovery derivation and chain watches. The policy is an
account property, independent of signing custody, and must survive reopen.

## Details

- Persist one immutable, non-null `NoChainSync` boolean for every SQL account.
  Existing accounts migrate to false.
- Carry the value through existing SQL account creation and read paths. All
  callers that do not explicitly request the policy continue writing false.
- Reject an attempt to change the value after account creation.
- Keep accountless raw imports chain-synchronized and leave wallet-level
  watch-only and signing capability unchanged.
- Public request, result, receiving, and recovery behavior remains in Tasks
  392, 393, 399, and 452.
- Do not add an account kind, discovery enum, custody flag, or modern-kvdb
  format.

## Suggested fix

Add the smallest two-dialect schema and Store conversion needed to persist the
boolean. Column and query organization may follow existing SQL conventions;
only the durable value and immutability are contractual.

## Verification

- Migrate an existing SQLite and PostgreSQL wallet and verify every account
  reopens with `NoChainSync=false`.
- Create and reopen accounts with both values and verify exact round trips.
- Prove null and update attempts cannot produce mutable or missing policy.
- Verify ordinary creation paths explicitly retain false and no public chain
  behavior changes in this task.

## Definition of done

Every SQL account has one immutable `NoChainSync` value that survives migration
and reopen, without changing custody, public selection, recovery, or chain
behavior.

[t340]: ../archive/tasks/340-durable-database-identity.md
[t387]: ../archive/tasks/387-account-creation-identity-adr.md
