# Task 288: Make Deprecated Imports Store-Mirror Safe

Status: Archived. Completed under the refined decision.

Archive note: the refined decision was to restore deprecated imports to pure
kvdb behavior, not to make SQL mirrors atomic. The SQL mirror calls from
`deprecated.go` are gone, and the deprecated constructor pins
`DBBackendKVDB`. Verification passed:

- `git grep -n "NewImportedAddress" HEAD -- wallet/deprecated.go` returned no
  matches
- `go test ./wallet -run 'TestManagerCreateRejectsLoadedWallet|TestManagerCreateRejectsStartedWallet|TestManagerCreateReleasesGuardOnFailure|TestResolveScanTargets|TestStoreScanHorizonsTargeted|TestGetUtxo|TestFilterEligibleOutputsExcludesNonSpendable|TestImportPrivateKeyKVDBSucceeds'`

Source review: `.lgtm/reviews/gpt-review-tasks.md`, finding 5.

## Finding

Several deprecated import paths mutate the legacy address manager before
mirroring the imported address or script into the SQL runtime store. If the SQL
mirror later fails, the legacy state has already changed and the wallet can end
up split-brained: waddrmgr sees the import, but the SQL store that drives scan
targets and ownership filtering does not.

Affected paths include legacy P2SH script import, public-key import, taproot
script import, and related deprecated private-key/script import flows.

## Rationale

The SQL-backed runtime paths read scan targets and ownership data from the
Store. A legacy-only import on a SQL wallet can leave funds invisible after
restart or recovery. The current mirror writes improve coverage for successful
imports, but they do not make the multi-store mutation safe on failure.

Full atomicity across bbolt and SQL is not available, so the fix needs to avoid
known SQL failures before the legacy write and make any remaining mirror failure
recoverable or idempotently repairable.

## Detailed Fix Plan

1. Inventory every deprecated import path that writes waddrmgr and should be
   visible through the Store:

   - P2SH redeem script import;
   - public-key import;
   - taproot script import;
   - private-key import;
   - any legacy RPC wrappers that call the same helpers.

2. Extract mirror payload construction before the legacy write. This should
   compute deterministic values such as scope, address type, script pubkey,
   public key bytes, encrypted private key, and encrypted script.

3. Add a non-mutating store validation path, or equivalent shared validation
   helper, that checks SQL invariants before waddrmgr is mutated. It should
   catch deterministic failures such as unsupported address type, missing secret
   material for spendable-wallet imports, malformed script payloads, and known
   duplicate semantics.

4. Run this prevalidation for non-kvdb runtime backends before the legacy
   `walletdb.Update`. Keep kvdb exempt because the kvdb Store is the legacy
   manager adapter and a mirror write would duplicate the same mutation.

5. Make the mirror write idempotent. If the legacy import reports a duplicate,
   verify that the Store already has the expected row and return success only
   when the existing SQL row matches the import payload.

6. Decide how to handle unexpected mirror failures after the legacy write. The
   preferred durable fix is to add a reconciliation path that can repair missing
   SQL mirror rows from legacy import state on startup or before scan target
   loading. If reconciliation is too large, document the remaining non-atomic
   failure mode and limit this task to deterministic prevalidation.

7. Keep notification registration after both stores agree, so callers do not get
   address notifications for an import that the Store cannot see.

## Tests

- Add tests that force each deterministic SQL validation failure and assert the
  legacy address manager was not mutated.
- Add duplicate-import tests for each deprecated path proving duplicate legacy
  imports either verify an existing matching Store row or repair a missing one.
- Add restart/scan-target tests showing SQL-backed wallets re-watch deprecated
  imports after reload.
- If a reconciliation path is added, test it against a deliberately legacy-only
  imported address/script.

## Verification

- `go test ./wallet -run 'Test.*Import.*Deprecated|Test.*Imported.*Scan'`
- `go test ./wallet/internal/db/...`
- DB integration tests for SQLite and PostgreSQL if SQL queries or schema are
  changed.

## Acceptance Criteria

- Deprecated import paths cannot return a deterministic SQL mirror error after
  already mutating waddrmgr.
- Successful deprecated imports are visible through the Store on SQL wallets.
- Duplicate imports are idempotent across both legacy and SQL state.
- Any remaining unavoidable cross-store failure mode is either repaired by a
  reconciliation path or explicitly documented with tests around the supported
  behavior.

## DECISION (2026-06-17 — supersedes the plan above)

Maintainer decision: deprecated import methods are **kvdb-only**. We will NOT
support SQL for these deprecated methods at all. This reframes the task from
"make the mirror safe" to "make deprecated imports kvdb-only and remove the SQL
mirror."

- **Reject** every deprecated import (P2SH redeem-script, public-key,
  taproot-script, private-key) on non-kvdb backends — return an error pointing
  callers at the modern account-scoped import API. The existing `isKVDB` checks
  that currently *skip the mirror* become *"deprecated import requires kvdb."*
- **Remove** the SQL mirror back-writes entirely (the `w.store.NewImportedAddress`
  calls): drop `fb9f2513a` (P2SH mirror) and the deprecated-import portions of
  `41c8ad246` (pubkey/taproot) and `c1a9d8d6c` (privkey) — all in `use-sql-store`,
  so the fold is top-of-stack.
- **Safe because** the syncer takes its watch-set from the `db.Store` interface;
  on a kvdb wallet the kvdb adapter surfaces `waddrmgr`'s managed addresses, so a
  deprecated import is watched through the store with no mirror. On SQL backends
  deprecated imports are simply rejected.
- Finding #5's split-brain dissolves by construction; the earlier
  "mirror P2SH / store-back deprecated imports" review fixes are intentionally
  reverted. See memory `feedback-no-sql-legacy-mirroring`.

### Refinement (2026-06-17, no per-method reject)

Deprecated methods are kvdb-only by the wallet's construction/config, so NO per-method "reject on non-kvdb" is needed — just REMOVE the mirror back-writes (restore pure-kvdb behavior). Backend exclusivity is enforced once, at a **config gate** in the final branch: if both a kvdb and a SQL backend are configured, return an error (a new-wallet config rule). The old/deprecated wallet is kvdb-only by construction and cannot reach the SQL store, so deprecated imports never see SQL.
