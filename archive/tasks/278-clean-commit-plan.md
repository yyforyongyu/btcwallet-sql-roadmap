# Task 278: Clean Commit Plan

Status: Archived. Completed and superseded by task 279 for the more granular
normalized identity cutover split plan.

## Outcome

The stack was restructured into the best achievable 7-commit series while
preserving a tree-byte-identical tip and buildable intermediate commits. The
original 13-commit plan was partially infeasible because the normalized identity
cutover is atomic in the current implementation: the new schema columns are
`NOT NULL` with no defaults, legacy inserts cannot satisfy them before the code
cutover, and normalized-only inserts cannot satisfy the retained legacy columns
before the schema cleanup. The shared `data_types.go` reshape and generated
`sqlc` hunks also interleave account, address, tx, and utxo changes.

Final commit structure:

1. `docs: add normalized identity ADR`
2. `db/sql: quote account number arg`
3. `wallet+db: use normalized identity`
4. `db/itest: add identity fixtures`
5. `db/itest: cover account identity`
6. `db/itest: cover address identity`
7. `db/itest: cover tx utxo identity`

## Goal

Restructure `review-base-fixes..fix-import-schema` into a reviewable,
bisectable commit series with smaller functional commits and clear subsystem
boundaries.

## Current Issues

- `88ba8cd80 docs: add ADR 0013 normalized account and address identity`
  mixes documentation with SQL query source changes.
- `5c1123815 db/sql: union account and address schema (additive)` is not
  bisectable because new `addresses.scope_id NOT NULL` constraints land before
  existing inserts populate `scope_id`.
- `48318974b db: cut account and address store over to normalized identity` is
  too large to review and is not bisectable because normalized-only inserts land
  before legacy `addresses.account_id NOT NULL` constraints are dropped.
- `ce0a31e6a db/itest: cover normalized account and address identity` is a
  large test-only commit that should be split by behavior under test.
- Several subjects exceed the preferred 50-character subject line budget.

## Line Budget

- Target at most 700 hand-authored changed lines per functional commit.
- Allow up to 900 hand-authored changed lines for symmetric SQL schema commits
  that must touch both PostgreSQL and SQLite.
- Keep generated `sqlc` files in the same commit as their SQL source, but
  exclude generated output from the review line budget.
- Split test-only commits when they exceed roughly 800-1,000 changed lines.

## Proposed Commit Plan

1. `docs: add normalized identity ADR`
   - ADR 0013, ADR index, and ADR 0012 update only.

2. `db/sql: quote account number arg`
   - Move the `sqlc.arg('account_number')` query fix here.
   - Include generated `sqlc` files only if regeneration changes them.

3. `db/sql: add normalized account schema`
   - Add normalized account schema while keeping legacy account columns.
   - Existing account insert paths must still satisfy all constraints.

4. `db/sql: add normalized address schema`
   - Add normalized address schema while keeping legacy address columns.
   - Existing address insert SQL must populate new `scope_id`,
     `script_type_id`, and `is_derived` values.
   - Do not switch to normalized-only inserts yet.

5. `db: add identity value types`
   - Add shared normalized identity structs, enum conversion helpers, and
     focused unit tests.
   - Do not flip store interfaces yet.

6. `wallet+db: use account identity`
   - Cut account store APIs/adapters over to normalized account identity.
   - Include kvdb, PostgreSQL, SQLite, account SQL queries, wallet account
     manager updates, and focused tests.
   - Continue dual-writing legacy schema where needed.

7. `wallet+db: use address identity`
   - Cut address store APIs/adapters over to normalized address identity.
   - Include address SQL queries, kvdb/PostgreSQL/SQLite address adapters,
     wallet address manager updates, and focused tests.
   - Continue dual-writing legacy `account_id`, `type_id`, branch, and index.

8. `wallet+db: update tx utxo identity joins`
   - Move tx/utxo queries and adapters to normalized identity joins.
   - Include `tx_creator`, `utxo_manager`, PSBT/signing test fallout, and
     focused tests.

9. `db/sql: drop legacy identity schema`
   - Drop legacy account/address columns and constraints.
   - Simplify inserts/queries to normalized-only behavior.
   - Regenerate `sqlc` in the same commit.

10. `db/itest: add identity fixtures`
    - Split shared DB integration fixture refactors into their own commit.

11. `db/itest: cover account identity`
    - Account creation, lookup, list, rename, and invariant coverage.

12. `db/itest: cover address identity`
    - Address creation/listing/lookup coverage, including imported vs derived
      identity shape and SQLite-specific identity checks.

13. `db/itest: cover tx utxo identity`
    - Tx/UTXO identity join coverage and corruption regression tests.

## Verification

- Every commit from the schema introduction onward must pass:
  `go test ./wallet/internal/db/... ./wallet`.
- Final DB integration test commits must pass:
  `go test -tags itest ./wallet/internal/db/itest`.
- The final range must pass:
  `git show --check --pretty=short review-base-fixes..fix-import-schema`.

## Acceptance Criteria

- Each commit has a clear subsystem prefix and scope.
- The docs commit is docs-only.
- Schema commits remain additive until runtime code has been cut over.
- Runtime commits dual-write legacy columns until the cleanup commit drops them.
- No intermediate commit fails the package-level wallet/DB tests.
- Generated `sqlc` changes are paired with the SQL source changes that produced
  them.
