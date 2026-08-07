# Task 279: Split Normalized Identity Cutover Further

Status: Archived.

Resolution: Superseded by Task 293, which replaces the earlier vertical-slice
plan with an additive lower-to-upper cutover and destructive upper-to-lower
cleanup plan.

## Goal

Restructure the large `wallet+db: use normalized identity` commit into smaller,
reviewable, preferably buildable vertical slices. The goal is not to resurrect
the original fully additive 13-commit plan from task 278, but to avoid squashing
all account, address, tx, UTXO, SQL, adapter, and wallet-call-site changes into
one commit.

## Current Commit Under Review

Current commit:
`8454ca16b wallet+db: use normalized identity`.

Current size:

- Full diff: 92 files, 5,343 insertions, 3,276 deletions.
- Excluding generated `sqlc`: 78 files, 3,975 insertions, 2,302 deletions.

Important subareas:

- Account SQL and generated account code: about 1,653 changed lines.
- Address SQL and generated address code: about 1,688 changed lines.
- Tx/UTXO SQL and generated code: about 750 changed lines.
- Account Go adapters/callers/tests: about 603 changed lines.
- Address Go adapters/callers/tests: about 2,276 changed lines.
- Tx/UTXO Go adapters/callers/tests: about 1,981 changed lines.

## Revised Assessment

The task-278 conclusion was directionally right that the original 13-commit
additive plan was too optimistic, but it was too broad to say that the cutover
cannot be split.

The real constraint is narrower:

- A pure layer split like "all SQL first, all Go later" will not stay buildable.
- A schema removal cannot be separated from the query and generated-code changes
  that stop referring to the removed fields.
- A query signature or result-shape change cannot be separated from the Go
  adapters that consume generated `sqlc` structs.
- Generated `sqlc` files should stay with the SQL source change that produced
  them.

However, the account, address, and tx/UTXO identity changes are not one
inseparable unit. They can be split as vertical slices and staged so that unused
schema additions land before behavioral cutovers, while destructive schema
cleanup lands only after code no longer uses the old fields.

## Actual Schema Dependencies

### Accounts Before Cutover

Before `8454ca16b`, `accounts` encoded both account shapes in one table:

- `account_number INTEGER` was nullable.
- `origin_id INTEGER NOT NULL` distinguished derived vs imported accounts.
- `imported_key_count INTEGER NOT NULL DEFAULT 0` tracked raw imported keys.
- `account_origins` was a lookup table with `derived` and `imported` rows.

This shape let derived and imported accounts share the same row, with nullable
fields indicating shape.

### Accounts After Cutover

After `8454ca16b`, account shape is normalized:

- `accounts.is_derived BOOLEAN NOT NULL` is the shape marker.
- `derived_accounts(account_id, scope_id, account_number)` stores BIP44 account
  numbers.
- Imported xpub accounts have an `accounts` row and no `derived_accounts` child.
- `account_origins`, `accounts.account_number`, `accounts.origin_id`, and
  `accounts.imported_key_count` are removed.

Useful split point:

- Adding `derived_accounts` and supporting indexes can be isolated as unused
  SQL scaffolding.
- Removing `account_number`/`origin_id` cannot happen until all queries and Go
  adapters read account numbers through `derived_accounts`.

### Addresses Before Cutover

Before `8454ca16b`, `addresses` encoded both address shapes in one table:

- `account_id INTEGER NOT NULL` owned every address.
- `type_id INTEGER NOT NULL` held the script/address type.
- `address_branch` and `address_index` were nullable.
- Imported addresses had `address_branch IS NULL` and `address_index IS NULL`.
- Triggers maintained `accounts.imported_key_count` for imported addresses.

### Addresses After Cutover

After `8454ca16b`, address shape is normalized:

- `addresses.scope_id INTEGER NOT NULL` stores wallet-local scope identity.
- `addresses.is_derived BOOLEAN NOT NULL` is the shape marker.
- `addresses.script_type_id INTEGER NOT NULL` replaces `type_id`.
- `derived_addresses` stores account ownership and BIP44 path data.
- Raw imported addresses have an `addresses` row and no `derived_addresses`
  child.
- Legacy `account_id`, `type_id`, `address_branch`, and `address_index` are
  removed from `addresses`.

Useful split point:

- Adding the new parent columns in nullable/defaulted form and adding
  `derived_addresses` can be isolated as unused SQL scaffolding.
- Removing legacy address columns cannot happen until address create/read code,
  tx joins, and UTXO joins no longer use them.

## What Can Be Pure SQL

These can be pure SQL plus generated `sqlc` output, with no hand-written Go
adapter changes, if they do not change query signatures used by Go:

- Add unused normalized child tables such as `derived_accounts`.
- Add unused supporting indexes and FKs that do not constrain current writes.
- Add new parent columns only if current writes can still succeed, meaning the
  new columns must be nullable or have safe defaults until the write cutover.
- Remove legacy columns after all query and Go users are already gone.
- Clean up SQL comments and docs wording, provided generated comment changes are
  either included or intentionally avoided.

These cannot be pure SQL in a buildable history:

- Changing `CreateDerivedAccount`, `CreateImportedAccount`,
  `CreateDerivedAddress`, or `CreateImportedAddress` parameters/results.
- Changing `GetAccount`, `ListAccounts`, `GetAddress`, `ListAddresses`,
  tx-detail, or UTXO query result shapes.
- Removing a column that any generated query or Go adapter still references.
- Replacing `origin_id` semantics with `is_derived` semantics in live queries.

## Proposed Commit Plan

### 1. `docs: add normalized identity ADR`

Keep the current docs-only commit.

Rationale:

- It establishes the design vocabulary before implementation.
- It is already clean after task 278.

Verification:

- Docs only; no package test needed beyond normal final checks.

### 2. `db/sql: quote account number arg`

Keep the extracted SQL argument quoting fix.

Rationale:

- This is an independent sqlc correctness/style fix.
- It should not be hidden inside the identity cutover.

Verification:

- Run the narrow package tests if generated output changes.

### 3. `docs: clean up identity wording`

Move non-behavioral wording cleanup here.

Scope:

- ADR wording that clarifies derived vs imported shape.
- SQL comments that only rename conceptual language, if they do not change
  query semantics.
- Avoid changing query names or result fields in this commit.

Rationale:

- The user-visible vocabulary cleanup is reviewable independently.
- It prevents comment churn from obscuring semantic SQL/Go changes.

Verification:

- `git show --check --pretty=short` for the commit.

### 4. `db/sql: add derived account scaffolding`

Add account normalization scaffolding without using it yet.

Scope:

- Add `derived_accounts` table.
- Add any supporting unique indexes needed for future FKs, such as an
  `(accounts.id, accounts.scope_id)` uniqueness guarantee.
- Do not remove `account_origins`, `accounts.account_number`,
  `accounts.origin_id`, or `accounts.imported_key_count`.
- Do not make current account inserts write new required fields.
- Include regenerated `sqlc` models if schema generation changes.

Rationale:

- This creates the new normalized table in isolation.
- Current Go code can continue using the legacy account shape.
- Reviewers can validate the child-table constraints separately from the Go
  cutover.

Important constraint:

- Do not add triggers that require `accounts.is_derived` unless this commit also
  adds a safe value for current writes. A trigger requiring a field that current
  writes do not populate will break tests.

Verification:

- `go test ./wallet/internal/db/... ./wallet`.

### 5. `wallet+db: write normalized account identity`

Teach account creation to populate normalized account identity while preserving
legacy columns for later consumers.

Scope:

- Add `accounts.is_derived` in a form current and new writes can satisfy.
- Update `CreateDerivedAccount` to insert the parent `accounts` row and the
  `derived_accounts` child row.
- Update `CreateImportedAccount` to insert an account with no
  `derived_accounts` child row.
- Keep legacy `account_number` and `origin_id` populated if any current query
  still reads them.
- Update pg/sqlite account create adapters and focused account create tests.
- Include generated `accounts.sql.go`, `db.go`, `querier.go`, and `models.go`
  changes produced by the SQL.

Rationale:

- Writes are the first behavioral cutover.
- Keeping legacy columns populated allows existing readers, address queries,
  tx queries, and UTXO queries to keep working during the transition.
- This commit should be much smaller than the current all-in-one cutover.

Verification:

- `go test ./wallet/internal/db/... ./wallet`.

### 6. `wallet+db: read normalized account identity`

Move account lookup/list/rename code to normalized account identity.

Scope:

- Update `GetAccount`, `ListAccounts`, account info conversion, and account
  rename queries to read shape from `accounts.is_derived` and account number
  from `derived_accounts`.
- Update `wallet/internal/db/accountstore_accountinfo.go` and tests.
- Update kvdb account adapter behavior to match the normalized contract.
- Update `wallet/account_manager.go` only where account store contract changes
  require it.
- Keep legacy account columns in schema for address/tx/UTXO users until they
  are migrated.

Rationale:

- Read-side account identity is reviewable separately from write-side account
  creation.
- Keeping legacy schema fields avoids forcing address and tx/UTXO query changes
  into this commit.

Verification:

- `go test ./wallet/internal/db/... ./wallet`.

### 7. `db/sql: add address identity scaffolding`

Add address normalization scaffolding without using it yet.

Scope:

- Add new address parent columns in transitional form:
  `scope_id`, `is_derived`, and `script_type_id`.
- Add `derived_addresses` table and supporting indexes/FKs that do not reject
  current writes.
- Keep legacy `addresses.account_id`, `type_id`, `address_branch`, and
  `address_index` untouched.
- Include generated models if sqlc emits them.

Rationale:

- This makes the address schema reviewable separately.
- It avoids mixing DDL, address adapters, tx/UTXO joins, and wallet callers in
  one commit.

Important constraint:

- New parent columns must not be `NOT NULL` without defaults while old address
  insert queries are still active. The old insert queries do not provide these
  values.

Verification:

- `go test ./wallet/internal/db/... ./wallet`.

### 8. `wallet+db: write normalized address identity`

Teach address creation to populate normalized address identity while preserving
legacy columns.

Scope:

- Update derived address creation to write the parent `addresses` row with
  `scope_id`, `is_derived`, and `script_type_id`.
- Insert `derived_addresses` child rows for derived addresses.
- Update imported address creation to write raw imported rows with no child row.
- Continue populating legacy address columns while they remain in the schema.
- Update pg/sqlite/kvdb address create adapters and focused creation tests.

Rationale:

- Write-side address cutover is distinct from read-side address conversion.
- Keeping legacy fields populated keeps tx/UTXO joins and existing address reads
  working until their own commits.

Verification:

- `go test ./wallet/internal/db/... ./wallet`.

### 9. `wallet+db: read normalized address identity`

Move address reads and address conversion helpers to normalized identity.

Scope:

- Update `GetAddress`, `ListAddresses`, and script-pubkey resolution queries to
  join through `derived_addresses` and `derived_accounts`.
- Update `addresses_common.go` and address conversion tests.
- Update pg/sqlite/kvdb address read/list adapters.
- Update `wallet/address_manager.go` and address tests only where result shape
  changes require it.
- Keep tx/UTXO queries on the legacy address columns for now if still possible.

Rationale:

- Address conversion is one of the largest chunks in the current commit.
- Splitting read conversion from write conversion makes the review tractable.

Verification:

- `go test ./wallet/internal/db/... ./wallet`.

### 10. `wallet+db: use normalized tx utxo joins`

Move transaction and UTXO ownership queries to normalized identity.

Scope:

- Update transaction and UTXO SQL to resolve ownership through normalized
  address/account identity.
- Update pg/sqlite generated code and adapters for tx rollback and UTXO get/list
  operations.
- Update kvdb tx/UTXO adapter contract behavior where needed.
- Update `wallet/tx_creator.go`, `wallet/utxo_manager.go`, and focused tests.

Rationale:

- Tx/UTXO joins are downstream of account and address identity.
- This should not be mixed with schema scaffolding or address conversion.

Verification:

- `go test ./wallet/internal/db/... ./wallet`.

### 11. `db/sql: drop legacy account identity`

Remove old account identity fields after all code has moved off them.

Scope:

- Drop `account_origins`.
- Drop `accounts.account_number`, `accounts.origin_id`, and
  `accounts.imported_key_count`.
- Make `accounts.is_derived` final and non-null if it was transitional.
- Add final account-shape triggers, such as the derived-parent trigger.
- Regenerate sqlc.

Rationale:

- This is the destructive account cleanup.
- It should be pure SQL/generated if commits 5, 6, 9, and 10 removed all users.

Verification:

- `go test ./wallet/internal/db/... ./wallet`.

### 12. `db/sql: drop legacy address identity`

Remove old address identity fields after all code has moved off them.

Scope:

- Drop `addresses.account_id`, `type_id`, `address_branch`, and
  `address_index`.
- Make `addresses.scope_id`, `is_derived`, and `script_type_id` final and
  non-null if they were transitional.
- Remove imported-key-count triggers tied to legacy address shape.
- Add final address-shape triggers, such as the derived-parent trigger.
- Regenerate sqlc.

Rationale:

- This is the destructive address cleanup.
- It should be pure SQL/generated if commits 8, 9, and 10 removed all users.

Verification:

- `go test ./wallet/internal/db/... ./wallet`.

### 13. `wallet+db: tighten identity cleanup tests`

Keep focused unit test adjustments close to the behavior they verify.

Scope:

- Move account-only unit tests into commits 5 and 6 where practical.
- Move address-only unit tests into commits 8 and 9 where practical.
- Move tx/UTXO unit tests into commit 10 where practical.
- Use this commit only for tests that genuinely span the whole normalized
  identity model.

Rationale:

- The current cutover commit mixes implementation and large test rewrites.
- Reviewers should see each behavior change next to its tests.

Verification:

- `go test ./wallet/internal/db/... ./wallet`.

### 14. `db/itest: add identity fixtures`

Keep the existing fixture split from task 278.

Rationale:

- Fixture churn is easier to review before behavioral itests.

Verification:

- `go test -tags itest ./wallet/internal/db/itest`.

### 15. `db/itest: cover account identity`

Keep account identity integration coverage isolated.

Rationale:

- Account behavior can be reviewed without address and tx/UTXO scenarios.

Verification:

- `go test -tags itest ./wallet/internal/db/itest`.

### 16. `db/itest: cover address identity`

Keep address identity integration coverage isolated.

Rationale:

- Address behavior has its own derived/raw-imported invariants.

Verification:

- `go test -tags itest ./wallet/internal/db/itest`.

### 17. `db/itest: cover tx utxo identity`

Keep tx/UTXO identity join integration coverage isolated.

Rationale:

- Tx/UTXO ownership joins are downstream of account and address identity.

Verification:

- `go test -tags itest ./wallet/internal/db/itest`.

## Fallback Plan

If a proposed commit fails to compile because sqlc validation or generated
interfaces cross the slice boundary, merge only the minimal adjacent slice:

- If commit 5 cannot compile alone, merge commits 5 and 6 into
  `wallet+db: use normalized account identity`.
- If commit 8 cannot compile alone, merge commits 8 and 9 into
  `wallet+db: use normalized address identity`.
- If commit 11 cannot compile before tx/UTXO updates, move it after commit 10.
- If generated `db.go` or `querier.go` churn is too interleaved, include it with
  the earliest query-changing commit that requires regeneration, but keep the
  hand-authored SQL and Go changes split by behavior.

Avoid falling back all the way to one monolithic cutover unless the failing
boundary has been verified with `go test ./wallet/internal/db/... ./wallet` and
the error is documented in the commit message or task notes.

## Line Budget Guidance

- Target under 700 hand-authored changed lines per functional commit.
- Allow larger SQL/generated commits when both pg and sqlite must change in
  lockstep.
- Exclude generated `sqlc` output from review budget, but keep it in the same
  commit as the source SQL.
- Split tests by behavior under test, not by backend.

## Verification Strategy

For every buildable implementation commit:

- Run `go test ./wallet/internal/db/... ./wallet`.
- Run `git show --check --pretty=short <commit>`.

For SQL source changes:

- Regenerate with `make sql` on a clean tree when Docker/tooling is available.
- If `make sql` is not available, at least run package tests and note that full
  SQL lint/codegen verification remains pending.

For final integration coverage:

- Run `go test -tags itest ./wallet/internal/db/itest`.
- Run the sqlite and postgres DB itest targets when Docker/Postgres are
  available.

## Acceptance Criteria

- The final tree remains equivalent to the current normalized identity tip,
  except for intentional bug fixes from review findings.
- Every intermediate commit should compile and pass package-level wallet/DB
  tests unless a documented fallback decision says otherwise.
- Account schema changes are separable from address schema changes where
  practical.
- Address schema changes are separable from tx/UTXO join changes where
  practical.
- Wording/comment cleanup is not mixed with semantic SQL or Go changes.
- Destructive schema cleanup lands after code no longer references the removed
  fields.
- Generated `sqlc` files are never edited manually and are paired with their SQL
  source changes.

## Resolution (2026-06-17): IRREDUCIBLE — SKIPPED (WONTFIX)

Verified against the current cutover commit `9c7f5f64b` "wallet+db: use normalized identity".

- The ADDITIVE scaffolding this task proposed (its commits 4 & 7 — derived_accounts/derived_addresses tables + nullable scope_id/is_derived columns + sqlc regen) is **already split off** as `99ccfd4c1` "db/sql: add normalized identity scaffolding" (269 ins, 0 del, no Go-contract change).
- The remainder `9c7f5f64b` is a genuine **atomic** schema cutover: it deletes/retypes the shared Go contract in `data_types.go` (`AccountOrigin` enum removed; `AccountNumber uint32→*uint32`; `Origin→IsImported`/`HasDerivationPath`), with in-place destructive migrations (`000005`/`000006`/`000008`) + lockstep-regenerated sqlc, all consumed by kvdb+pg+sqlite+wallet simultaneously. Every proposed seam (SQL-generated-leads, account/address/utxo vertical slices, read/write split) produces a non-compiling intermediate — refuted with specific symbols.
- Per the maintainer directive (attempt the split; skip if irreducible) — **SKIPPED**. Confirms [[feedback-stacked-restructure-cutover-irreducible]].
