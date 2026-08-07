# Task 293: Additive normalized identity cutover plan

> **Archived 2026-07-01:** done — the additive normalized-identity cutover landed in fix-import-schema (#1268).

Status: Open.

## Goal

Rewrite the `fix-import-schema` normalized identity changes into buildable,
reviewable commits using staged compatibility instead of replacement commits.
Each commit should compile. Prefer commits around 200-400 changed lines when
possible, but buildability has priority over the line budget.

## Strategy

Additions flow from lower layers to upper layers:

1. Add SQL/storage capability.
2. Add shared Go contract fields while retaining old fields.
3. Populate both old and new fields in backend adapters.
4. Migrate one wallet manager or consumer at a time.

Deletions flow from upper layers to lower layers:

1. Stop upper-layer code from reading or passing old fields.
2. Remove old shared Go contract fields after all consumers are gone.
3. Stop generated queries from selecting old columns.
4. Remove old SQL columns/tables/triggers last.

Avoid transition comments. Do not mention that a field is newly added, temporary,
kept for compatibility, or scheduled for removal. Field comments should describe
only the current contract, and only when the comment is required or useful. The
cleanup commits make the old shape disappear.

Avoid superficial helpers. Do not add thin helpers just to wrap pointer literals,
small struct construction, or one-line conversions. Prefer local variables or
inline code unless the helper removes real duplication across multiple tests or
encapsulates non-obvious behavior.

Preserve kvdb internals. The normalized schema is for SQL backends. kvdb is a
legacy format that should keep its existing buckets, account constants, and
imported-account semantics internally. kvdb changes should be limited to mapping
legacy data into the shared Store API shape required by SQL-wallet callers.

Generated `sqlc` output stays with the SQL source that produced it. Generated
diff size does not count against the review budget, but it still affects compile
boundaries.

## Field Inventory

Account identity changes:

- Add `accounts.is_derived`.
- Add `AccountInfo.AccountID *uint32`.
- Add `AccountInfo.IsImported bool`.
- Change `AccountInfo.AccountNumber` from `uint32` to `*uint32`.
- Remove `AccountInfo.Origin`.
- Remove `AccountOrigin`, `DerivedAccount`, and `ImportedAccount` after address
  and UTXO users are also migrated.
- Remove `account_origins`, `accounts.origin_id`, and
  `accounts.imported_key_count` last.

Address identity changes:

- Add `addresses.is_derived`.
- Add `addresses.script_type_id` alongside `type_id` before replacing it.
- Add `derived_addresses` while legacy `addresses.account_id`,
  `address_branch`, and `address_index` still exist.
- Add `AddressInfo.IsImported bool`.
- Add `AddressInfo.HasDerivationPath bool`.
- Change `AddressInfo.AccountID` from `uint32` to `*uint32`.
- Change `AddressInfo.AccountNumber` from `uint32` to `*uint32`.
- Change `ListAddressesQuery.AccountName` from `string` to `*string`.
- Change `ListAddressesQuery.Scope` from `KeyScope` to `*KeyScope`.
- Remove `AddressInfo.Origin` after consumers migrate.
- Remove legacy address columns last.

UTXO and derivation changes:

- Add `UtxoInfo.Spendable *bool` while keeping `UtxoInfo.Origin`.
- Remove `UtxoInfo.Origin` after wallet coin selection uses `Spendable`.
- Add `AddressDerivationParams.AccountID *uint32`.
- Add `AddressDerivationParams.DerivedAccountNumber *uint32` before removing
  `AddressDerivationParams.AccountNumber`.
- Remove `NewImportedAddressParams.Scope` after upper layers stop passing it.

## Proposed Commit Plan

### 1. `docs: add normalized identity ADR`

Keep the docs-only ADR commit and ADR index update, but rewrite ADR 13 before
implementation work proceeds.

Required ADR cleanup from review:

- Frame the DB model around whether identity is derivable, not the overloaded
  word imported. A wallet can be imported from a seed while its accounts are
  still derivable; an imported account xpub can derive child addresses; a raw
  imported script cannot derive child addresses.
- Keep user-facing imported terminology only where it describes the source of
  material, not the DB derivability invariant.
- Add a compact shape table covering wallet-derived accounts, imported-xpub
  accounts, derived child addresses, imported-xpub child addresses, and raw
  imported addresses.
- Be specific about which reads benefit from table splitting and why.
- Remove claims that UTXO/balance reads need corruption checks when write-time
  constraints already make invalid states impossible.
- Explain that the SQL schema does not expand kvdb's internal model; kvdb keeps
  legacy buckets and imported-account constants.

### 2. `db/sql: add account derived marker`

Add `accounts.is_derived` in pg/sqlite while keeping `origin_id`. Account create
queries write both fields. Existing account reads can continue using
`origin_id`.

Verification: `go test ./wallet/internal/db/... ./wallet`.

### 3. `db: expose account import flag`

Add `AccountInfo.IsImported` while keeping `AccountInfo.Origin`. Update pg,
sqlite, and kvdb account producers to populate both fields. Update focused DB
unit tests for the new field only where needed.

Do not add pointer helper functions for tests in this commit. Use local pointer
variables where needed.

Verification: `go test ./wallet/internal/db/... ./wallet`.

### 4. `wallet: use account import flag`

Switch `wallet/account_manager.go` and `wallet/account_manager_test.go` from
`Origin` to `IsImported`.

Verification: `go test ./wallet`.

### 5. `wallet: use import flag for tx accounts`

Switch `wallet/tx_creator.go` and `wallet/tx_creator_test.go` from `Origin` to
`IsImported` for account handling.

Verification: `go test ./wallet`.

### 6. `db: make account numbers optional`

Change `AccountInfo.AccountNumber` to `*uint32`. Update all account producers,
direct wallet consumers, and focused tests. Keep `Origin` and `IsImported` both
available.

Do not introduce generic `testUint32Ptr` style helpers unless the surrounding
test already has a comparable fixture helper with real behavior. Prefer:

```go
account := uint32(0)
info.AccountNumber = &account
```

Verification: `go test ./wallet/internal/db/... ./wallet`.

### 7. `db/sql: add address script type id`

Add `addresses.script_type_id` while keeping `type_id`. Address write queries
populate both. Reads can continue using `type_id`.

Verification: `go test ./wallet/internal/db/... ./wallet`.

### 8. `db/sql: add address derived marker`

Add `addresses.is_derived` while keeping legacy `account_id`, `address_branch`,
and `address_index`. Address write queries populate the marker.

Verification: `go test ./wallet/internal/db/... ./wallet`.

### 9. `db: expose address import and path flags`

Add `AddressInfo.IsImported` and `AddressInfo.HasDerivationPath` while keeping
`AddressInfo.Origin`. Update pg, sqlite, and kvdb address producers to populate
both old and new shapes.

Verification: `go test ./wallet/internal/db/... ./wallet`.

### 10. `wallet: use normalized address flags`

Switch `wallet/address_manager.go` and address-manager tests from `Origin` and
branch/index inference to `IsImported` and `HasDerivationPath`.

Verification: `go test ./wallet`.

### 11. `db: make address account ids optional`

Change `AddressInfo.AccountID` and `AddressInfo.AccountNumber` to pointers.
Update address producers, wallet consumers, and focused tests.

Verification: `go test ./wallet/internal/db/... ./wallet`.

### 12. `db/sql: add derived address table`

Add `derived_addresses` plus supporting indexes and foreign keys that do not
break existing writes. Keep legacy address ownership/path columns.

Verification: `go test ./wallet/internal/db/... ./wallet`.

### 13. `db/sql: dual-write derived addresses`

Derived address creation writes both the legacy address columns and the new
`derived_addresses` child row. Raw imported addresses continue to write the old
shape until listing/import callers are migrated.

Verification: `go test ./wallet/internal/db/... ./wallet`.

Do not add kvdb buckets or kvdb-only normalized persistence for this. kvdb should
derive any shared API fields from existing waddrmgr/legacy structures.

### 14. `db/sql: read derived address identity`

Switch address get/list/resolve queries and adapters to read account/path data
from `derived_addresses` and script type from `script_type_id`, while legacy
columns remain in schema.

Verification: `go test ./wallet/internal/db/... ./wallet`.

### 15. `db: support accountless address listing`

Change `ListAddressesQuery.AccountName` and `Scope` to pointers. Nil account
and nil scope list raw imported addresses. Update adapters and wallet callers.

Verification: `go test ./wallet/internal/db/... ./wallet`.

### 16. `wallet: omit imported address scope`

Stop upper layers from setting `NewImportedAddressParams.Scope`. Keep the field
in the shared type so lower layers still compile.

Verification: `go test ./wallet`.

### 17. `db: remove imported address scope param`

Delete `NewImportedAddressParams.Scope` after all upper-layer callers are off
it. Update store implementations and tests.

Verification: `go test ./wallet/internal/db/... ./wallet`.

### 18. `db/sql: check utxo wallet through address`

Change UTXO wallet-consistency triggers to use `addresses.wallet_id` rather than
joining through accounts. This prepares for accountless raw imports.

Verification: `go test ./wallet/internal/db/... ./wallet`.

### 19. `db: expose utxo spendability`

Add `UtxoInfo.Spendable` while keeping `UtxoInfo.Origin`. SQL backends can leave
it nil under wallet-level spendability. kvdb sets it where legacy mixed
watch-only state matters.

Do not add new kvdb buckets or indexes. Preserve existing kvdb storage and
compute the shared field from legacy account/address state.

Verification: `go test ./wallet/internal/db/... ./wallet`.

### 20. `wallet: use utxo spendability`

Switch `wallet/utxo_manager.go` and UTXO wallet tests from `Origin` to
`Spendable`/wallet-level rules.

Verification: `go test ./wallet`.

### 21. `db: pass normalized derivation identity`

Add `AddressDerivationParams.AccountID` and `DerivedAccountNumber` while keeping
`AccountNumber`. Update derivation callbacks and store implementations to pass
both shapes.

Verification: `go test ./wallet/internal/db/... ./wallet`.

### 22. `db: remove legacy derivation account number`

Remove `AddressDerivationParams.AccountNumber` after all callbacks use
`DerivedAccountNumber`.

Verification: `go test ./wallet/internal/db/... ./wallet`.

### 23. `db: remove account origin API`

Delete `AccountInfo.Origin` after account-manager and tx-account users are off
it. Keep `AccountOrigin` if address or UTXO APIs still use it.

Verification: `go test ./wallet/internal/db/... ./wallet`.

### 24. `db: remove address origin API`

Delete `AddressInfo.Origin` after address-manager and DB address tests are off
it.

Verification: `go test ./wallet/internal/db/... ./wallet`.

### 25. `db: remove utxo origin API`

Delete `UtxoInfo.Origin`. Delete `AccountOrigin`, `DerivedAccount`, and
`ImportedAccount` if no remaining Go users exist.

Verification: `go test ./wallet/internal/db/... ./wallet`.

### 26. `db/sql: stop selecting account origins`

Remove `origin_id` from account, address, and UTXO queries. Regenerate sqlc.
The SQL column still exists in migrations until the next cleanup commit.

Verification: `go test ./wallet/internal/db/... ./wallet`.

### 27. `db/sql: remove account origin schema`

Remove `account_origins` and `accounts.origin_id` from pg/sqlite migrations and
generated models after all queries stop using them.

Verification: `go test ./wallet/internal/db/... ./wallet`.

### 28. `db/sql: remove legacy account counters`

Remove `accounts.imported_key_count` and any remaining imported-key-count
triggers after raw imports no longer depend on account child rows.

Verification: `go test ./wallet/internal/db/... ./wallet`.

### 29. `db/sql: remove legacy address shape`

Remove old `addresses.account_id`, `type_id`, `address_branch`, and
`address_index`; make normalized address columns final; keep generated code with
the SQL source changes.

Verification: `go test ./wallet/internal/db/... ./wallet`.

### 30. `db/itest: cover account identity`

One larger integration-test commit for account identity, including account raw
fixture helpers required by the tests.

Verification: focused SQLite and Postgres account identity itests.

### 31. `db/itest: cover address identity`

One larger integration-test commit for address identity, including address raw
fixture helpers required by the tests.

Verification: focused SQLite and Postgres address identity itests.

### 32. `db/itest: cover tx utxo identity`

One larger integration-test commit for tx/UTXO identity and ownership joins,
including tx/UTXO raw fixture helpers required by the tests.

Verification: focused SQLite and Postgres tx/UTXO identity itests.

## Line Budget Guidance

- Target 200-400 changed lines per production commit where practical.
- Prefer a buildable 500-line commit over a non-buildable split.
- Generated sqlc output is allowed to exceed the budget when paired with the SQL
  source that generated it.
- Integration-test commits may be larger: account, address, and tx/UTXO coverage
  should each be self-contained and reviewable.
- Avoid comment-only churn in intermediate commits. If a comment does not help
  the final code after cleanup, do not add it.

## Verification Strategy

For every implementation commit:

```shell
go test ./wallet/internal/db/... ./wallet
```

For wallet-only commits:

```shell
go test ./wallet
```

For SQL commits:

```shell
make sql
git status --short
go test ./wallet/internal/db/... ./wallet
```

For final integration coverage:

```shell
go test -tags 'itest dev debug stdlog' ./wallet/internal/db/sqlite ./wallet/internal/db/itest -run '<focused case>'
go test -tags 'itest dev debug stdlog test_db_postgres' ./wallet/internal/db/pg ./wallet/internal/db/itest -run '<focused case>'
```

## Acceptance Criteria

- Every intermediate commit compiles.
- Destructive schema cleanup happens only after no generated query or Go adapter
  references the legacy field.
- Account changes are not forced into address or tx/UTXO commits except where a
  compile boundary proves they are inseparable.
- Address changes are not forced into tx/UTXO commits except where ownership
  joins require it.
- Intermediate commits avoid transitional prose and unnecessary comment churn.
- The final tree matches the intended normalized identity tree, aside from
  documented review fixes.
