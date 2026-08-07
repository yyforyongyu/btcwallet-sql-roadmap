# Task 281: Enforce Stack Branch Commit Quota

Status: Archived. Superseded by task 283.

Archive note: this plan was completed as a planning artifact, but its exact
branch split is now stale. Task 283 replaces the old `impl-secret-backend`
split with a cleaner `impl-secret-store` plus `use-sql-runtime` split.

## Goal

Restructure the SQL-wallet stack so each branch stays within the branch commit
quota while preserving existing PR branch names.

Quota:

- Preferred: at most 10 commits per branch.
- Hard cap: at most 15 commits per branch.

Branch naming rule:

- If a branch already has a PR, keep that branch name so the PR number remains
  meaningful.
- It is acceptable to move commits between existing PR branches as long as the
  PR branch names remain in the stack.
- Create new branch names only when the existing branch slots are insufficient
  to keep every branch at or below 15 commits.

## Current Stack

The stack is conceptually based on old `sql-wallet` commit `8cca7b619`, not the
currently advanced local `sql-wallet` branch tip.

Current lineage:

```text
8cca7b619
  └ review-base-fixes      #1267
    └ fix-import-schema    #1268
      └ prep-runtime-1     #1262
        └ prep-runtime-2   #1263
          └ impl-runtime-store    #1256
            └ impl-runtime-store-2  #1257
              └ impl-runtime-store-3  #1258
                └ impl-runtime-store-4
                  └ impl-secret-store
                    └ use-sql-store
```

Current commit counts:

| Branch | Parent Range | Commits | Status |
|---|---|---:|---|
| `review-base-fixes` | `8cca7b619..review-base-fixes` | 2 | OK |
| `fix-import-schema` | `review-base-fixes..fix-import-schema` | 7 | OK |
| `prep-runtime-1` | `fix-import-schema..prep-runtime-1` | 23 | Over cap |
| `prep-runtime-2` | `prep-runtime-1..prep-runtime-2` | 7 | OK |
| `impl-runtime-store` | `prep-runtime-2..impl-runtime-store` | 22 | Over cap |
| `impl-runtime-store-2` | `impl-runtime-store..impl-runtime-store-2` | 18 | Over cap |
| `impl-runtime-store-3` | `impl-runtime-store-2..impl-runtime-store-3` | 9 | OK |
| `impl-runtime-store-4` | `impl-runtime-store-3..impl-runtime-store-4` | 6 | OK |
| `impl-secret-store` | `impl-runtime-store-4..impl-secret-store` | 20 | Over cap |
| `use-sql-store` | `impl-secret-store..use-sql-store` | 18 | Over cap |

## Target Stack

Target lineage after restructuring:

```text
8cca7b619
  └ review-base-fixes          #1267    2 commits
    └ fix-import-schema        #1268    7 commits
      └ prep-runtime-1         #1262    9 commits
        └ prep-runtime-2       #1263   14 commits
          └ prep-runtime-3      new     7 commits
            └ impl-runtime-store      #1256   12 commits
              └ impl-runtime-store-2  #1257   10 commits
                └ impl-runtime-store-3  #1258  8 commits
                  └ impl-runtime-store-4       10 commits
                    └ impl-runtime-store-5  new 15 commits
                      └ impl-secret-backend  new 12 commits
                        └ impl-secret-store       8 commits
                          └ use-sql-runtime   new  3 commits
                            └ use-sql-store        15 commits
```

Total branches after base: 14.

New branches needed:

- `prep-runtime-3`
- `impl-runtime-store-5`
- `impl-secret-backend`
- `use-sql-runtime`

Existing branch names reused:

- `review-base-fixes`
- `fix-import-schema`
- `prep-runtime-1`
- `prep-runtime-2`
- `impl-runtime-store`
- `impl-runtime-store-2`
- `impl-runtime-store-3`
- `impl-runtime-store-4`
- `impl-secret-store`
- `use-sql-store`

## Target Ranges

| Target Branch | Range | Commits | Notes |
|---|---|---:|---|
| `review-base-fixes` | `8cca7b619..abaef7756` | 2 | Existing PR #1267 |
| `fix-import-schema` | `abaef7756..69bfc4e11` | 7 | Existing PR #1268 |
| `prep-runtime-1` | `69bfc4e11..4d8d1bea7` | 9 | First part of old `prep-runtime-1` |
| `prep-runtime-2` | `4d8d1bea7..09cd9f03b` | 14 | Second part of old `prep-runtime-1` |
| `prep-runtime-3` | `09cd9f03b..785c8aefd` | 7 | Old `prep-runtime-2` content |
| `impl-runtime-store` | `785c8aefd..8ac2ef726` | 12 | First runtime-store implementation slice |
| `impl-runtime-store-2` | `8ac2ef726..0ee53e612` | 10 | Scan-batch tests + rollback sync-tip |
| `impl-runtime-store-3` | `0ee53e612..e1853c005` | 8 | Runtime store core syncer routing |
| `impl-runtime-store-4` | `e1853c005..5560b2698` | 10 | Filtered block + scan routing |
| `impl-runtime-store-5` | `5560b2698..c15e51934` | 15 | Runtime store final cleanup/router slice |
| `impl-secret-backend` | `c15e51934..21ad1a16d` | 12 | Keyvault + Store secret backend contracts |
| `impl-secret-store` | `21ad1a16d..8688fb758` | 8 | Wallet-facing secret-store routing |
| `use-sql-runtime` | `8688fb758..6c2083bfe` | 3 | Runtime SQL config/open/backend support |
| `use-sql-store` | `6c2083bfe..30ff15ea3` | 15 | Final SQL-store wiring and review fixes |

## Commit Group Rationales

### `prep-runtime-1`

Range: `fix-import-schema..4d8d1bea7`.

Theme:

- `ListSyncedBlocks` types, SQL, and kvdb/sqlite/pg implementations.
- `DeleteExpiredLeases` on `Store`.
- `RollbackToBlock` sync-state rewind.

Rationale:

- These are prep-runtime store capabilities needed before scan/tx batch work.
- The branch stays below the preferred 10-commit target.

### `prep-runtime-2`

Range: `4d8d1bea7..09cd9f03b`.

Theme:

- `ListOutputsToWatch`.
- watched bare-multisig coverage.
- `ApplyTxBatch` request types and backend implementations.
- `ApplyTxBatch` conformance coverage.

Rationale:

- This keeps the old PR #1263 branch name while moving the heavier second half
  of old `prep-runtime-1` into it.
- The branch is 14 commits, above preferred but within hard cap.

### `prep-runtime-3`

Range: `09cd9f03b..785c8aefd`.

Theme:

- Scan horizon extension helper types and shared helpers.

Rationale:

- Old `prep-runtime-2` content needs a new branch slot after `prep-runtime-2`
  is reused for the second half of old `prep-runtime-1`.

### `impl-runtime-store`

Range: `785c8aefd..8ac2ef726`.

Theme:

- kvdb scan horizon handlers.
- shared scan-batch helpers.
- sqlite/pg scan batch implementation.
- `ApplyScanBatch` store interface.

Rationale:

- Keeps existing PR #1256 branch name.
- Branch is 12 commits, over preferred but under hard cap.

### `impl-runtime-store-2`

Range: `8ac2ef726..0ee53e612`.

Theme:

- ApplyScanBatch unit/conformance tests.
- Rollback sync-tip fix and itest.

Rationale:

- Keeps existing PR #1257 branch name.
- Exactly at preferred cap.

### `impl-runtime-store-3`

Range: `0ee53e612..e1853c005`.

Theme:

- Pass runtime store into syncer.
- Route startup setup, sync-tip updates, rollback reads/writes, rebroadcast
  reads, and relevant tx notifications through the store.

Rationale:

- Keeps existing PR #1258 branch name.
- Below preferred cap.

### `impl-runtime-store-4`

Range: `e1853c005..5560b2698`.

Theme:

- Filtered block batch routing and tests.
- Store scan-batch parameter construction and scan write routing.
- Scan-horizon account-name stamping.

Rationale:

- Keeps existing no-PR branch name.
- Exactly at preferred cap.

### `impl-runtime-store-5`

Range: `5560b2698..c15e51934`.

Theme:

- Store scan data loading.
- Targeted rescan durable identity resolution.
- Drop legacy DB fallbacks.
- Make syncer Store mandatory.
- Remove dead legacy DB helpers.
- Route manager master fingerprint reads through Store.

Rationale:

- A new branch is required because `impl-runtime-store-4` plus these commits
  would exceed the 15-commit hard cap.
- This is the final runtime-store cleanup slice.

### `impl-secret-backend`

Range: `c15e51934..21ad1a16d`.

Theme:

- Keyvault key type/auth params.
- Legacy manager vault shim.
- `GetAccountSecret` SQL/query/types/backend implementations.
- kvdb legacy lifecycle/signer/scan-target adapters.
- Assemble kvdb `db.Store` backend.

Rationale:

- A new branch is required because existing `impl-secret-store` currently has
  20 commits.
- This isolates backend contracts and adapters from wallet-facing routing.

### `impl-secret-store`

Range: `21ad1a16d..8688fb758`.

Theme:

- Construct legacy keyvault shim.
- Expand vault interface.
- Route auth, wallet lifecycle, signer pubkey/key lookups, and syncer scan
  targets through kvdb/store-backed paths.

Rationale:

- Keeps the existing no-PR `impl-secret-store` branch name for the wallet-facing
  secret-store integration layer.

### `use-sql-runtime`

Range: `8688fb758..6c2083bfe`.

Theme:

- Runtime store config.
- Runtime store selection/opening.
- SQL wallet backend itest support.

Rationale:

- A new branch is required because current `use-sql-store` has 18 commits.
- This isolates the base SQL runtime enablement from later review fixes and
  behavior corrections.

### `use-sql-store`

Range: `6c2083bfe..30ff15ea3`.

Theme:

- Review fixes from prior full-stack reviews.
- Lifecycle/create/import/signing fixes.
- Account schema mapping fix.
- P2SH redeem-script mirror.
- Confirm-reuse replay edge handling.
- Recovery lookahead by account xpub.

Rationale:

- Keeps the final no-PR `use-sql-store` branch name.
- Exactly at the hard cap of 15 commits.

## Verification Already Performed

Boundary compile checks passed with:

`go test -run '^$' ./wallet/internal/db/... ./wallet`

Checked tips included:

- `4d8d1bea7`
- `09cd9f03b`
- `785c8aefd`
- `8ac2ef726`
- `0ee53e612`
- `e1853c005`
- `5560b2698`
- `c15e51934`
- `21ad1a16d`
- `8688fb758`
- `6c2083bfe`
- current `use-sql-store` tip `30ff15ea3`

Additional DB package tests passed for earlier prep-runtime boundaries:

- `3b0fefd65`
- `454c0f76a`
- `09cd9f03b`
- `4d8d1bea7`

## Implementation Steps

1. Ensure the worktree is clean except for intentional `.lgtm/` task files.
2. Create or reset `prep-runtime-1` to `4d8d1bea7`.
3. Create or reset `prep-runtime-2` to `09cd9f03b`, based on
   `prep-runtime-1`.
4. Create `prep-runtime-3` at `785c8aefd`, based on `prep-runtime-2`.
5. Create or reset `impl-runtime-store` to `8ac2ef726`, based on
   `prep-runtime-3`.
6. Create or reset `impl-runtime-store-2` to `0ee53e612`, based on
   `impl-runtime-store`.
7. Create or reset `impl-runtime-store-3` to `e1853c005`, based on
   `impl-runtime-store-2`.
8. Create or reset `impl-runtime-store-4` to `5560b2698`, based on
   `impl-runtime-store-3`.
9. Create `impl-runtime-store-5` at `c15e51934`, based on
   `impl-runtime-store-4`.
10. Create `impl-secret-backend` at `21ad1a16d`, based on
    `impl-runtime-store-5`.
11. Create or reset `impl-secret-store` to `8688fb758`, based on
    `impl-secret-backend`.
12. Create `use-sql-runtime` at `6c2083bfe`, based on `impl-secret-store`.
13. Keep or reset `use-sql-store` to `30ff15ea3`, based on `use-sql-runtime`.
14. Re-run boundary compile checks.
15. Retarget PRs as needed so existing PR branch names keep their PR numbers.

## Acceptance Criteria

- Every branch has at most 15 commits over its immediate parent.
- Branches with existing PRs keep their branch names.
- New branch names are limited to the four required slots:
  `prep-runtime-3`, `impl-runtime-store-5`, `impl-secret-backend`, and
  `use-sql-runtime`.
- Each branch tip compiles with `go test -run '^$' ./wallet/internal/db/... ./wallet`.
- DB-focused branch tips pass `go test ./wallet/internal/db/...` where feasible.
- The final `use-sql-store` tip remains the intended full-stack tip.
