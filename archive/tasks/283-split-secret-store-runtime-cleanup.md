# Task 283: Split Secret Store From Runtime Cleanup

Status: Archived. Completed.

Archive note: current stack has no `impl-secret-backend` branch in the active
lineage. `impl-secret-store` contains only the 10 secret/key-material commits,
and `use-sql-runtime` contains the legacy kvdb runtime cleanup plus runtime SQL
enablement commits. Verification passed:

- `go test -run '^$' ./wallet/internal/db/... ./wallet` on `impl-secret-store`
- `go test ./wallet -run 'TestResolveDerivedPrivKeyFromStore'` on
  `impl-secret-store`
- `go test -run '^$' ./wallet/internal/db/... ./wallet` on `use-sql-runtime`
- `go test ./wallet -run 'TestResolveDerivedPrivKeyFromStore'` on
  `use-sql-runtime`

Historical check before completion:

- `impl-secret-backend` is still contained by `impl-secret-store`,
  `use-sql-runtime`, and `use-sql-store`.
- `impl-secret-store` still includes runtime cleanup commits, including
  `kvdb: add legacy signer derivation adapters`.
- `impl-secret-store` still includes `kvdb: assemble db.Store backend`.
- `impl-secret-store` still includes
  `wallet: route signer pubkey lookups through kvdb`.
- `impl-secret-store` still includes
  `wallet: route syncer scan targets through kvdb`.
- `impl-secret-backend` also includes `wallet: remove dead legacy DB helpers`,
  which is runtime cleanup rather than secret-store work.

## Goal

Reshape the SQL-wallet stack so `impl-secret-store` contains only the secret
and key-material work, while the legacy-kvdb runtime cleanup commits move to
`use-sql-runtime`.

This supersedes the stale `impl-secret-backend` split from task 281. Do not keep
`impl-secret-backend` as a separate branch unless a later branch quota check
proves a new branch slot is still required.

## Current Decision

`kvdb: add legacy signer derivation adapters` belongs on `use-sql-runtime`, not
`impl-secret-store`.

Reason:

- It only wraps existing legacy `walletdb`/`waddrmgr` signer reads behind the
  `kvdb.Store` boundary.
- It does not add SQL secret storage, account-secret schema, keyvault plumbing,
  or store-backed key derivation.
- It is needed so top-level wallet runtime code can stop opening `walletdb.View`
  directly.

## Keep In `impl-secret-store`

These commits are secret/key-material work and should remain together:

- `kvdb: add legacy manager vault shim`
- `db: add GetAccountSecret query`
- `db: add account secret types`
- `pg: implement GetAccountSecret`
- `sqlite: implement GetAccountSecret`
- `db: add GetAccountSecret to the store contract`
- `wallet: construct legacy keyvault shim`
- `wallet: route unlock and lock through keyvault`
- `wallet: add store-backed key derivation primitives`
- `wallet: route signer keys through Store on legacy miss`

## Move To `use-sql-runtime`

These commits are runtime cleanup or runtime enablement, not secret-store work:

- `wallet: remove dead legacy DB helpers`
- `kvdb: add legacy wallet lifecycle helpers`
- `kvdb: add legacy signer derivation adapters`
- `kvdb: add legacy scan target resolver`
- `kvdb: assemble db.Store backend`
- `wallet: route wallet lifecycle through kvdb`
- `wallet: route signer pubkey lookups through kvdb`
- `wallet: route syncer scan targets through kvdb`
- `wallet: add runtime store config`
- `wallet: select and open the runtime store`
- `wallet+itest: support SQL wallet backends`

## Expected Branch Shape

Target ordering near the top of the stack:

```text
impl-runtime-store-4
  └ impl-secret-store
    └ use-sql-runtime
      └ use-sql-store
```

Branch responsibilities:

- `impl-secret-store`: keyvault, account-secret storage, and store-backed signer
  private-key fallback.
- `use-sql-runtime`: legacy kvdb runtime adapters, wallet runtime store config,
  runtime store open/select logic, and SQL wallet backend test wiring.
- `use-sql-store`: final SQL-store behavior fixes and review fixes.

## Conflict Notes

Expect conflicts around signer routing because
`wallet: route signer keys through Store on legacy miss` currently sits after
`wallet: route signer pubkey lookups through kvdb` in the old stack.

Resolution principle:

- Keep the private-key fallback commit in `impl-secret-store`.
- Keep pubkey lookup routing and legacy derivation adapters in
  `use-sql-runtime`.
- Preserve the final behavior where legacy-backed accounts use the legacy path
  and store-only accounts can fall back to `GetAccountSecret`.

## Verification

After reshaping, run boundary checks at each affected branch tip:

- `go test -run '^$' ./wallet/internal/db/... ./wallet`
- Focused signer fallback tests:

  ```shell
  go test ./wallet -run 'TestResolveDerivedPrivKeyFromStore'
  ```

- `git show --check --pretty=short <range-or-commit>`

For the final `use-sql-store` tip, also re-run the focused review tests from
the latest stack review task before considering the split complete.

## Acceptance Criteria

- `impl-secret-backend` is retired or unused.
- `impl-secret-store` contains only secret/key-material commits.
- `use-sql-runtime` contains the legacy kvdb runtime adapters and runtime SQL
  enablement commits.
- Existing PR branch names are preserved where they already exist.
- Each affected branch stays under the 15-commit hard cap if possible.
- The final `use-sql-store` tree remains equivalent except for intentional
  conflict-resolution or review-fix changes.
