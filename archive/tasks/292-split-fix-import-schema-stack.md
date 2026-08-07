# Task 292: Split `fix-import-schema` commit stack

Status: Archived. Completed.

Archive note: PR `#1268` was rebuilt into an add/switch/remove compatibility
split stack, verified against `sql-wallet`, and force-pushed to
`upstream/fix-import-schema` from `a1c3ef682` to `ab3d345cb` with an explicit
lease. Final stack:

- `docs: add normalized identity ADR`
- `db/sql: add normalized identity schema`
- `db+wallet: consume normalized accounts`
- `db+wallet: consume normalized addresses`
- `db+wallet: consume normalized utxos`
- `db/sql: remove legacy identity shape`
- `db/itest: add identity fixtures`
- `db/itest: cover account creation`
- `db/itest: cover account listing`
- `db/itest: cover address identity`
- `db/itest: cover tx UTXO identity`

Verification passed:

- `git diff --check`
- ADR 13 line-length check
- `go test ./wallet/internal/db/... ./wallet -run '^$'`
- `go test ./wallet/internal/db/...`
- `go test ./wallet`
- `go test -tags=itest ./wallet/internal/db/itest`
- `make fmt-check`
- `make sqlc-check`
- `make lint-check`
- `scripts/check-each-commit.sh sql-wallet`

## Background

PR `#1268` currently targets `sql-wallet` from branch
`upstream/fix-import-schema`. The latest pushed head is `a1c3ef682`, whose
largest commit is:

```text
7596752cb db+wallet: normalize SQL identity
```

That commit is too large to review comfortably. It combines account identity,
address identity, UTXO/transaction identity, generated SQL code, DB adapters,
kvdb compatibility, wallet behavior, and unit-test updates.

The earlier decision to keep SQL, sqlc, and dependent adapters together was
based on one failed split: a SQL/sqlc-only commit did not build. That failure
does not justify flattening account, address, and UTXO identity changes into a
single implementation commit. The new split should be domain-based and
buildable at every commit.

## Current State

- Main PR worktree:
  `.worktrees/review-pr-1268-current`.
- Temporary rebuild worktree:
  `.worktrees/rebuild-pr-1268-split`.
- Temporary rebuild branch:
  `fix-import-schema-split-rebuild`.
- Useful old split reference:
  `backup/fix-import-schema-rework-before-stack-rewrite`.
- The old split reference contains separate SQL account, SQL address, SQL UTXO,
  wallet-consumption, and itest commits. It is a useful starting point, but it
  is not final because the current branch has later ADR cleanup and review
  fixes.
- Do not push another rewrite until the new stack passes per-commit checks.

## Goal

Rewrite the PR stack into smaller, reviewable, buildable commits while keeping
the final tree equivalent to the current pushed head, except for intentional ADR
reorganization and any explicitly documented hygiene fixes.

## ADR 13 Rewrite

Before changing implementation commits, rewrite
`docs/developer/adr/0013-normalized-account-address-identity.md` so the decision
record is organized by domain.

### Accounts Section

The `Accounts` section should explain:

- The final design: `accounts` is the stable account identity table.
- `accounts.account_number` is nullable.
- Wallet-derived accounts use `is_derived = true` and a non-null BIP44 account
  number.
- Imported xpub accounts use `is_derived = false` and a NULL account number.
- There is no `derived_accounts` table.
- Stable `account_id` is the durable identity for derived accounts and imported
  xpub accounts.
- `is_derived` is structural shape, not provenance.
- Account identity fields are immutable after creation.

The `Accounts` section should also document alternatives and tradeoffs:

- Rejected alternative: keep fake account number `0` for imported xpubs.
- Rejected alternative: split account numbers into `derived_accounts`.
- Accepted tradeoff: nullable `account_number` keeps the invariant row-local and
  avoids parent/child drift, while still removing fake identity.
- Pros: imported xpubs cannot collide with BIP44 account `0`; account lookup by
  number is derived-only; recovery can key by durable account ID.
- Cons: Go callers must treat account numbers as optional; imported account code
  must not collapse NULL to zero.

### Addresses Section

The `Addresses` section should explain:

- The final design: `addresses` is wallet-local script identity.
- `addresses` stores wallet ID, script pubkey, script type, imported public-key
  material, creation time, and `is_derived`.
- `addresses` does not store scope, account, branch, or index.
- `derived_addresses` stores account ownership plus branch and index.
- Derived address scope is resolved through
  `derived_addresses.account_id -> accounts.scope_id`.
- Raw imported addresses are accountless and scopeless `addresses` rows with no
  `derived_addresses` child.
- The reserved name `imported` is a wallet-facing compatibility alias only; it is
  not materialized as an SQL account row.

The `Addresses` section should also document alternatives and tradeoffs:

- Rejected alternative: keep raw imports under an SQL account named `imported`.
- Rejected alternative: keep nullable account/path columns on `addresses`.
- Accepted tradeoff: `derived_addresses` adds a join and requires consistent
  shape checks, but keeps raw imports from carrying fake account, scope, or path
  identity.
- Pros: raw imports are modeled honestly; derived address indexes can start from
  account/path facts; imported-xpub children remain derived addresses.
- Cons: address reads need joins; all read paths must enforce the same malformed
  parent/child shape checks.

### ADR Constraints

- Do not rewrite ADR 12. ADRs are moment-in-time decision records.
- ADR 13 may say it supersedes ADR 12's SQL imported-address bucket shape, but
  it should preserve ADR 12 as historical context.
- Keep the account/address case table, but place it where it supports the two
  domain sections instead of replacing the detailed rationale.
- Keep line lengths near 80 columns.

## Target Commit Stack

The target stack has nine commits:

```text
docs: add normalized identity ADR
db+wallet: normalize account identity
db+wallet: normalize address identity
db+wallet: normalize UTXO identity
db/itest: add identity fixtures
db/itest: cover account creation
db/itest: cover account listing
db/itest: cover address identity
db/itest: cover tx UTXO identity
```

If the account/address/UTXO split exposes an unavoidable compile-time dependency,
stop and report the exact dependency. Do not silently flatten the implementation
back into one large commit.

## Detailed Commit Contents

### 1. `docs: add normalized identity ADR`

Contains ADR 13 plus the ADR index update only.

ADR 13 should be rewritten into two main sections: `Accounts` and `Addresses`.
Each section explains final design, alternatives, pros/cons, and rationale.
ADR 12 must have no diff.

Expected files:

- `docs/developer/adr/0013-normalized-account-address-identity.md`.
- `docs/developer/adr/README.md`.

### 2. `db+wallet: normalize account identity`

Contains all account-specific production and unit-test changes needed to build.

Include:

- Account migrations and account queries for PostgreSQL and SQLite.
- Generated account sqlc files needed by those queries.
- `AccountInfo` optional account number changes.
- Account ID exposure and account row shape validation.
- PostgreSQL and SQLite account adapters.
- kvdb account mapping changes.
- Wallet account-manager behavior.
- Account-focused unit tests.

Do not include address list behavior, raw imported address handling, UTXO
filtering, or coin selection unless strictly required to compile.

Likely file groups:

- `wallet/internal/sql/*/migrations/000005_accounts.*.sql`.
- `wallet/internal/sql/*/queries/accounts.sql`.
- `wallet/internal/sql/*/sqlc/accounts.sql.go`.
- Account-related generated sqlc model/interface files when required by sqlc.
- `wallet/internal/db/accountstore_*.go`.
- `wallet/internal/db/pg/accountstore_*.go`.
- `wallet/internal/db/sqlite/accountstore_*.go`.
- `wallet/internal/db/kvdb/accountstore_*.go`.
- `wallet/account_manager.go` and account-focused tests.

### 3. `db+wallet: normalize address identity`

Contains all address-specific production and unit-test changes needed to build.

Include:

- Address migrations and address queries for PostgreSQL and SQLite.
- Generated address sqlc files needed by those queries.
- `AddressInfo` identity changes.
- `AddressDerivationParams` changes.
- PostgreSQL and SQLite address adapters.
- kvdb address mapping changes.
- Raw imported address list semantics.
- `ListAddresses` helper/lint cleanup.
- Wallet address-manager behavior.
- Address-focused unit tests.

Do not include UTXO/balance query changes or `tx_creator` coin-selection behavior
unless strictly required to compile.

Likely file groups:

- `wallet/internal/sql/*/migrations/000006_addresses.*.sql`.
- `wallet/internal/sql/*/queries/addresses.sql`.
- `wallet/internal/sql/*/sqlc/addresses.sql.go`.
- Address-related generated sqlc model/interface files when required by sqlc.
- `wallet/internal/db/addresses_common.go`.
- `wallet/internal/db/pg/addresses.go`.
- `wallet/internal/db/sqlite/addresses.go`.
- `wallet/internal/db/kvdb/addressstore.go`.
- `wallet/address_manager.go` and address-focused tests.

### 4. `db+wallet: normalize UTXO identity`

Contains UTXO, balance, transaction ownership, and coin-selection identity
changes.

Include:

- UTXO and transaction query changes for PostgreSQL and SQLite.
- Generated UTXO and transaction sqlc files.
- UTXO wallet-consistency trigger updates.
- PostgreSQL and SQLite UTXO adapters.
- kvdb UTXO mapping changes.
- Wallet UTXO-manager behavior.
- `tx_creator` imported-alias handling.
- Wallet-layer raw-import UTXO compatibility filtering.
- UTXO, balance, and coin-selection unit tests.

Likely file groups:

- `wallet/internal/sql/*/migrations/000008_utxos.up.sql`.
- `wallet/internal/sql/*/queries/utxos.sql`.
- `wallet/internal/sql/*/queries/transactions.sql`.
- `wallet/internal/sql/*/sqlc/utxos.sql.go`.
- `wallet/internal/sql/*/sqlc/transactions.sql.go`.
- `wallet/internal/db/utxo_store_common.go`.
- `wallet/internal/db/pg/utxostore_*.go`.
- `wallet/internal/db/sqlite/utxostore_*.go`.
- `wallet/internal/db/kvdb/utxostore*.go`.
- `wallet/internal/db/kvdb/txstore.go`.
- `wallet/utxo_manager.go`.
- `wallet/tx_creator.go` and related tests.

### 5. `db/itest: add identity fixtures`

Contains shared integration-test fixtures only.

Include:

- Backend fixture helpers.
- Raw SQL identity mutation helpers.
- PostgreSQL and SQLite fixture setup.
- Shared helpers needed by later itest commits.

Likely files:

- `wallet/internal/db/itest/fixtures_common_test.go`.
- `wallet/internal/db/itest/fixtures_pg_test.go`.
- `wallet/internal/db/itest/fixtures_sqlite_test.go`.

### 6. `db/itest: cover account creation`

Contains account creation and account invariant integration tests.

Include:

- Derived account creation coverage.
- Imported xpub account creation coverage.
- Nullable account number assertions.
- Reserved imported alias assertions.
- Malformed account shape checks.
- `accountNumberNotNil` helper rename, if it belongs with these tests.

Likely files:

- `wallet/internal/db/itest/accountstore_createderived_test.go`.
- `wallet/internal/db/itest/accountstore_createinvariants_test.go`.
- `wallet/internal/db/itest/accountstore_getaccount_test.go` when creation
  setup needs immediate lookup assertions.
- `wallet/internal/db/itest/accountstore_helpers_test.go`.

### 7. `db/itest: cover account listing`

Contains account listing and rename integration tests.

Include:

- `ListAccounts` behavior.
- `GetAccountByName` and `GetAccountByNumber` listing-related coverage.
- Rename behavior.
- Imported xpub account listing.
- Account-number nil/non-nil assertions.

Likely files:

- `wallet/internal/db/itest/accountstore_listaccounts_test.go`.
- `wallet/internal/db/itest/accountstore_renameaccount_test.go`.
- Account helper updates only if not already in the creation commit.

### 8. `db/itest: cover address identity`

Contains address identity integration tests.

Include:

- Derived address rows.
- Imported-xpub child addresses.
- Raw imported address rows.
- Accountless/scopeless raw imports.
- SQLite identity/rowid immutability.
- Malformed derived-address shape rejection.
- Address listing behavior.

Likely files:

- `wallet/internal/db/itest/address_store_test.go`.
- `wallet/internal/db/itest/sqlite_identity_test.go`.
- Address fixture/helper updates only if not already in the fixture commit.

### 9. `db/itest: cover tx UTXO identity`

Contains transaction and UTXO integration tests.

Include:

- UTXO enrichment coverage.
- Account filter behavior.
- Wallet-wide nil filters including raw imports.
- Malformed UTXO/address join rejection.
- Imported alias behavior where applicable.
- Transaction owned-output identity coverage.

Likely files:

- `wallet/internal/db/itest/tx_store_corruption_test.go`.
- `wallet/internal/db/itest/tx_utxo_store_test.go`.

## Commit Boundary Rules

### `db+wallet: normalize account identity`

Include account-specific changes only:

- Account migrations and account query changes for PostgreSQL and SQLite.
- Generated account sqlc changes needed by those queries.
- Account public data type changes, including optional account numbers.
- Account row shape validation.
- PostgreSQL, SQLite, and kvdb account adapters.
- Wallet account-manager behavior and account-focused unit tests.

Do not include address or UTXO behavior unless required to compile. If a helper
must be shared, keep only the minimal shared type/helper needed by the account
commit.

### `db+wallet: normalize address identity`

Include address-specific changes only:

- Address migrations and address query changes for PostgreSQL and SQLite.
- Generated address sqlc changes needed by those queries.
- `AddressInfo` and address derivation identity shape changes.
- PostgreSQL, SQLite, and kvdb address adapters.
- Raw imported address listing semantics.
- Wallet address-manager behavior and address-focused unit tests.

Do not include UTXO filtering, balance behavior, or coin-selection behavior
unless required to compile.

### `db+wallet: normalize UTXO identity`

Include UTXO and transaction ownership changes:

- UTXO and transaction query changes for PostgreSQL and SQLite.
- Generated UTXO and transaction sqlc changes.
- PostgreSQL, SQLite, and kvdb UTXO enrichment/filtering adapters.
- Wallet UTXO-manager behavior.
- `tx_creator` imported-alias coin-selection behavior.
- UTXO, balance, and coin-selection unit tests.

This is also the place for wallet-layer compatibility filtering of the reserved
`imported` alias for raw imported UTXOs, if it is not already needed earlier.

## Integration Test Commits

Keep the integration coverage split by behavior:

```text
db/itest: add identity fixtures
db/itest: cover account creation
db/itest: cover account listing
db/itest: cover address identity
db/itest: cover tx UTXO identity
```

If fixture changes can be moved closer to the first test that uses them without
hurting readability, do so. Otherwise keep the existing fixture commit if it is
the most reviewable option.

## Execution Plan

1. Work in `.worktrees/rebuild-pr-1268-split` until the rewritten stack is
   verified.
2. Use `backup/fix-import-schema-rework-before-stack-rewrite` as the split
   starting point.
3. Apply the current branch's later review fixes onto the appropriate domain
   commits:
   - ADR 12 restoration and ADR 13 rewrite into the docs commit.
   - `ListAddresses` lint helper cleanup into the address commit.
   - raw imported UTXO alias mapping into the UTXO commit.
   - `accountNumberNotNil` helper rename into the account itest commit.
4. Rebase/autosquash so there are no temporary or fixup commits.
5. Compare the final tree against current pushed head `a1c3ef682` and document
   intentional differences.
6. Run verification.
7. Push with an explicit `--force-with-lease` only after verification passes.

## Verification

Required before push:

```shell
git diff --check
go test ./wallet
go test ./wallet/internal/db/...
go test -tags=itest ./wallet/internal/db/itest
make fmt-check
make sqlc-check
make lint-check
scripts/check-each-commit.sh sql-wallet
```

The most important check is `scripts/check-each-commit.sh sql-wallet`; every
commit in the rewritten stack must build. If that check fails, fix the split or
stop with the exact failure and proposed boundary change.

## Push Safety

Before pushing:

```shell
git fetch upstream fix-import-schema
git ls-remote upstream refs/heads/fix-import-schema
```

Then push only with an explicit lease against the observed remote head:

```shell
git push --force-with-lease=refs/heads/fix-import-schema:<remote-head> \
    upstream HEAD:fix-import-schema
```

## Acceptance Criteria

- ADR 13 is organized into account and address sections with design,
  alternatives, pros/cons, and rationale.
- ADR 12 has no PR diff.
- The large `db+wallet: normalize SQL identity` commit is gone.
- Implementation changes are split into account, address, and UTXO domain
  commits unless an unavoidable build dependency is documented.
- No fixup, temp, or WIP commits remain.
- Per-commit verification passes.
- The pushed branch matches the verified local head.

## Implementation Findings

Attempted on 2026-06-26 in temporary worktrees:

- `.worktrees/rebuild-pr-1268-domain`.
- `.worktrees/rebuild-pr-1268-adjusted`.
- `.worktrees/rebuild-pr-1268-split`.

The original nine-commit domain plan is not buildable without additional
temporary compatibility commits:

- An account-first commit fails SQL generation because account queries already
  reference `derived_addresses`, while the address schema/query changes are not
  present yet.
- The same account-first state also leaves existing address and UTXO queries
  referencing removed `origin_id` columns.
- The older SQL-only split under
  `backup/fix-import-schema-rework-before-stack-rewrite` also fails per-commit
  verification at `db/sql/pg: normalize account schema`: generated shared sqlc
  files reference address query methods that are not generated until the address
  commit.
- Splitting production code from unit-test updates fails per-commit compilation
  because `AccountInfo.AccountNumber` changes from `uint32` to `*uint32`, and
  the old tests still use struct literals such as `AccountNumber: 0`.
- A temporary compatibility shim would need to make `AccountNumber` accept both
  `uint32` and `*uint32` in struct literals. Go cannot model that cleanly
  without changing the field to `any`, which would be worse than keeping the
  production/API/test cutover atomic.

Current safe conclusion:

- ADR 13 can be rewritten independently.
- The SQL/account/address/UTXO production cutover and the unit tests that compile
  against the new public structs must remain in one buildable commit unless the
  branch accepts explicit temporary compatibility commits.
- The existing DB integration test commits can remain split.

Do not archive this task until the user decides whether to accept a smaller
verified split with one production/API/unit-test cutover commit, or to introduce
temporary compatibility commits solely for commit hygiene.
