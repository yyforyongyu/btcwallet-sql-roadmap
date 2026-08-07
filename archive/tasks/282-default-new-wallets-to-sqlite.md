## Task 282: Default New Wallets To SQLite

> **Archived 2026-07-01:** superseded by task 295 — the sqlite-default flip now lives in the wallet-default-sqlite branch (per task 294/295).

Status: Open. Implementation appears present on `use-sql-runtime`, but package
verification is not clean yet.

Latest check:

- `go test ./wallet -run 'TestDBConfig|TestConfigValidate'` passed on
  `use-sql-runtime`.
- `go test ./wallet` failed on `use-sql-runtime` in syncer tests
  `TestInitChainSyncWatchesUnspentForSpends` and
  `TestInitChainSync_NotifyBlocksError`.

Likely stack-placement fix: move or split the relevant parts of
`a12dcd652 review(syncer): restore btcd block ntfns + SQL sync fixes` earlier,
because that later `use-sql-store` commit makes the final wallet package tests
pass.

### Context

New library-created wallets should use the SQL runtime store by default. The
default should be SQLite, while explicit `DBBackendKVDB` remains available for
opening existing kvdb-only wallets for compatibility.

This is a wallet-library behavior change only. It does not include CLI/RPC
wallet creation, `btcwallet --create`, or kvdb-to-SQL migration/backfill.

### Desired Placement

Fold this into the planned `use-sql-runtime` branch split, ideally in the
commit that introduces runtime DB config:

- `wallet: add runtime store config`

Do not add it as another top-level commit on the current oversized
`use-sql-store` tip unless the stack is no longer being rewritten.

### Requirements

- Treat an unset `DB.Backend` as `DBBackendSQLite`.
- Keep explicit `DBBackendKVDB` as the compatibility path for existing kvdb
  wallets.
- Keep explicit `DBBackendPostgres` unchanged.
- Keep `DB.KVDB.DBPath` required for now because the legacy compatibility store
  is still opened for manager/key paths.
- If the backend resolves to SQLite and `DB.SQLite.DBPath` is empty, derive a
  deterministic default path from `DB.KVDB.DBPath`, likely `wallet.sqlite` in
  the same directory.
- Update docs/comments that currently say zero-value config selects kvdb.
- Update wallet tests/helpers so tests that require legacy behavior set
  `Backend: DBBackendKVDB` explicitly.

### Non-Goals

- No kvdb-to-SQL migration/backfill.
- No CLI/RPC default changes.
- No itest default changes unless the branch owner explicitly wants library and
  harness defaults to move together.

### Verification

- Run `go test ./wallet` after the change.
- If the runtime config change touches database-store behavior, also run the
  narrow relevant DB tests, but SQL generation should not be needed.

### Notes

Existing kvdb-only wallets must be opened with `DB.Backend=DBBackendKVDB` until
a separate migration project exists. Leaving `DB.Backend` unset intentionally
selects the SQLite runtime path for new wallets.

## Resolution (2026-06-17)

- **SQLite-default impl (Item 1): DONE/verified** — `runtime_store_config.go::withDefaults()` defaults unset Backend→SQLite, derives `wallet.sqlite` from `KVDB.DBPath`, keeps kvdb/postgres compat; introduced in the right commit (`wallet: add runtime store config`). No work.
- **Config gate (Item 3): ACCEPTED AS ALREADY-SATISFIED** — the literal "error if both kvdb+sql configured" is not viable: the legacy compat store is mandatory, so every default SQLite wallet sets `KVDB.DBPath` (the SQLite path derives from it). The intent (new=SQL / old=kvdb-only / deprecated kvdb-only) is already enforced structurally: `Backend` is a scalar selector, `deprecated.go` pins kvdb, and `DBConfig.Validate()` already rejects SQL sub-config under the kvdb backend. No new gate code. A true config-level kvdb/SQL split would require removing the mandatory compat store — a separate, larger project (deferred).
- **use-sql-runtime package-test cleanliness (Item 2): DOING the commit-move** — extract the ~15-line `NotifyBlocks` initChainSync fix (+3 syncer_test.go changes) from the use-sql-store syncer-repair commit into use-sql-runtime, leaving the SQL-coupled remainder on use-sql-store. Handled in the Phase-4 reshape.
