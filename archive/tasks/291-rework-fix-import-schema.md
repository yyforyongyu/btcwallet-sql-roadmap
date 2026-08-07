# Task 291: Rework `fix-import-schema` normalized identity commits

> **Archived 2026-07-01:** done — fix-import-schema (#1268) was reworked and is green on the merged track.

Status: Open.

## Background

The inclusive range on `fix-import-schema` from
`4d2222e4a05e8862c75a8d575aa3f06efcacf398` to
`6c78b5676d040e11f6804865246ba166d8ffa1fc` is too large to review cleanly:

```text
4d2222e4a wallet+db: use normalized identity
da869ff75 db/itest: add identity fixtures
df7a823bd db/itest: cover account identity
77525c1ee db/itest: cover address identity
6c78b5676 db/itest: cover tx utxo identity
```

The main offender is `4d2222e4a`, which mixes the final normalized-identity
cutover across SQL schema/queries, generated sqlc code, shared db types,
PostgreSQL/SQLite adapters, kvdb compatibility, wallet behavior, and unit tests.

Important nuance: the parent already has transitional normalized scaffolding
(`derived_accounts`, `derived_addresses`, composite FK indexes). This task is
the final cutover away from legacy identity columns and origin enums, not a
from-scratch schema introduction.

## Goal

Resplit the range into reviewable, buildable commits. The user is comfortable
using up to 15 commits for this branch.

## Proposed Split

1. `db: expose normalized account identity`
   - Add `AccountInfo.AccountID *uint32`.
   - Make `AccountInfo.AccountNumber *uint32`.
   - Replace public account `Origin` usage with `IsImported`.
   - Add account row shape validation helpers.
   - Keep SQL schema and wallet behavior out unless required to compile.

2. `db: expose normalized address identity`
   - Make `AddressInfo.AccountID *uint32`.
   - Make `AddressInfo.AccountNumber *uint32`.
   - Add `AddressInfo.IsImported` and `HasDerivationPath`.
   - Replace address `Origin` usage in shared converters.
   - Add address/account shape validation helpers.

3. `db: pass normalized derivation identity`
   - Add `AddressDerivationParams.AccountID *uint32`.
   - Rename/replace `AccountNumber` with `DerivedAccountNumber *uint32`.
   - Pass nil for imported-xpub account derivation.
   - Add shared unit coverage for derived vs imported-xpub derivation identity.

4. `db/sql: cut account queries to child identity`
   - Remove account-origin query usage.
   - Read account numbers from `derived_accounts`.
   - Use `accounts.is_derived` for account shape.
   - Update account lookup/list/rename queries for pg and sqlite.
   - Regenerate sqlc for account-query changes.

5. `db/sql: cut account writes to child identity`
   - Create derived account parent rows plus `derived_accounts` child rows.
   - Create imported-xpub account rows without `derived_accounts` child rows.
   - Update pg/sqlite create-derived and create-imported adapters.
   - Preserve reserved imported-alias behavior.

6. `db/sql: cut address schema to normalized parents`
   - Drop legacy address ownership/path columns from canonical schema.
   - Make `addresses.scope_id`, `is_derived`, and `script_type_id` canonical.
   - Keep HD path/account ownership in `derived_addresses`.
   - Keep raw imported addresses as wallet/scope address rows with no account.
   - Regenerate sqlc for schema/model changes.

7. `db/sql: cut address queries to child identity`
   - Update address get/list/resolve queries to join `derived_addresses`,
     `accounts`, and `derived_accounts`.
   - Add/use raw imported address listing by wallet and scope.
   - Update pg/sqlite address read adapters.

8. `db/sql: cut address writes to child identity`
   - Create derived address parent rows plus `derived_addresses` child rows.
   - Create raw imported address rows without materializing the imported alias
     as an account row.
   - Update pg/sqlite derived/imported address write adapters.

9. `db/sql: enforce normalized identity immutability`
   - Account identity immutability trigger for `id`, `wallet_id`, `scope_id`,
     and `is_derived`.
   - Address identity immutability trigger for `id`, `wallet_id`, `scope_id`,
     and `is_derived`.
   - Reject updates to `derived_accounts` and `derived_addresses` rows.
   - Include matching down migrations.

10. `db/sql: normalize tx owned-output joins`
    - Update transaction owned-output/input queries to rejoin `addresses` for
      wallet ownership.
    - Preserve tx read/write semantics while avoiding legacy account joins.
    - Regenerate sqlc for tx query changes.

11. `db/sql: normalize utxo ownership joins`
    - Update `InsertUtxo`, `GetUtxoByOutpoint`, `ListUtxos`, and `Balance` to
      join through normalized address/account identity.
    - Use `UtxoAddressShape` validation.
    - Add/update pg/sqlite UTXO adapters.
    - Update UTXO wallet-consistency trigger to validate through
      `addresses.wallet_id`.

12. `db/kvdb: align normalized account mapping`
    - Imported accounts expose nil `AccountNumber`.
    - Balance merge keys use internal waddrmgr account numbers, not public nil
      account numbers.
    - Preserve kvdb account lookup/list behavior under the new contract.

13. `db/kvdb: align normalized address and utxo mapping`
    - Raw imported addresses report no account identity.
    - Keep synthetic address IDs local to kvdb.
    - Align UTXO enrichment fields with SQL: account name, address type,
      script flag, lock flag, and key scope.

14. `wallet: consume normalized store identity`
    - Update account manager mapping for `IsImported` and nil account numbers.
    - Update address manager mapping for `HasDerivationPath` and nil account
      numbers.
    - Avoid fabricating account 0 for imported-xpub children.
    - Update change-source and coin-selection account-name routing.
    - Update wallet unit tests from the current large commit.

15. `db/itest: cover normalized identity cutover`
    - Move only the necessary fixtures into this commit.
    - Cover account identity/shape, address identity/shape, SQLite rowid alias
      immutability, UTXO account-name filters, and malformed join rejection.
    - If this commit is still too large, split it into account/address/UTXO
      itest commits and keep the branch at or below the 15-commit hard cap by
      folding the smallest production-adapter commits above only if they remain
      logically inseparable and buildable.

## Fixture Guidance

Do not keep `da869ff75 db/itest: add identity fixtures` as a standalone fixture
dump. Put raw SQL fixture helpers beside the tests that first need them:

- Account raw mutators with account identity tests.
- Address raw mutators with address identity tests.
- UTXO/tx helpers with UTXO identity tests.
- Backend-specific SQLite rowid helpers with SQLite-only identity tests.

## Verification

Run the narrowest checks after each slice where practical:

```shell
go test -run '^$' ./wallet/internal/db/... ./wallet
go test ./wallet/internal/db
go test ./wallet/internal/db/kvdb
go test ./wallet
```

For SQL changes:

```shell
make sql
git status --short
```

For integration coverage:

```shell
go test -tags itest ./wallet/internal/db/itest -run '<focused identity cases>'
go test -tags 'itest test_db_postgres' ./wallet/internal/db/itest -run '<focused identity cases>'
```

Before finishing, verify the final tree is content-equivalent to the original
range tip, unless intentional review fixes are made and documented separately.
