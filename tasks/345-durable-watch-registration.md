# Task 345 — persist SQL account discovery mode

Component: **AddressManager and SQL Store**.

Dependencies: **[Task 340][t340]**.

## Rationale

The planned allocation contract needs a durable key-only mode. Keeping that
mode only in an allocation request would lose it on restart and conflict with
account-wide recovery and ingestion policy.

## Details

- Choose immutable `watch-chain` or `key-only` discovery mode when the account
  is created. Allocations inherit it and reject any attempted override.
- Add the SQL storage migration first: backfill every preexisting account as
  `watch-chain`, then make discovery mode present and immutable for all account
  rows. Prove the backfill and immutability after reopen before new creation
  paths depend on the field.
- Key-only locators remain derivable/signable but never become wallet credits,
  recovery targets, balances, or watched outputs.
- During transaction ingestion, classify an output matching a key-only locator
  as non-owned. Retain the transaction's other credits, debits, status, and sync
  update; one key-only output must not reject the whole batch. Key-only accounts
  never create watch obligations.
- Reject ordinary receiving-address allocation for key-only accounts. Their
  next durable locator is allocated through [Task 344][t344]'s count-one key operation,
  and an existing locator remains derivable through [Task 337][t337]. Neither path uses
  an API that promises a receivable wallet address.
- Treat every accountless raw address or script import as `watch-chain` in the
  backend-neutral mode contract.
- Land the SQL discovery-mode column, preexisting-account backfill,
  immutability proof, regenerated SQL, and both-dialect reopen tests as the
  artifact consumed by [Task 335][t335].
- Keep registration obligations, retry, pending results, chain calls, and
  historical-scan isolation in [Task 364][t364].

## Suggested fix

Add the SQL discovery-mode column, backfill every existing account as
`watch-chain`, and make the field required and immutable. Route account
creation, allocation, ingestion, and accountless raw imports through one
backend-neutral mode type without adding delivery machinery in this task.

## Verification

Migration and reopen tests on SQLite and PostgreSQL prove every preexisting
account becomes `watch-chain` before the field is made immutable. Account
creation and allocation tests reject mode overrides after mutation. Ingestion
tests prove a key-only match creates no credit, balance, watched output, or
watch obligation while retaining the transaction's other effects. Receiving
APIs reject key-only accounts before allocating a locator, and raw imports are
classified as `watch-chain`.

## Definition of done

- SQL stores an immutable discovery mode, backfills every existing account as
  `watch-chain`, and preserves the backend-neutral mode contract after reopen.
- Live watch delivery, retry, raw-import obligations, and Neutrino scan
  isolation are owned separately by
  [Task 364](364-durable-watch-registration.md).

[t335]: 335-kvdb-to-sql-migration.md
[t337]: 337-imported-xpub-derivation.md
[t340]: 340-durable-database-identity.md
[t344]: 344-bulk-address-allocation.md
[t364]: 364-durable-watch-registration.md
