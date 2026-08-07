# Task 294: Reorganize use-sql-runtime / use-sql-store — prep on kvdb, sqlite-default last

Status: DONE (2026-07-01) — implemented (route B + option A) and pushed to both
remotes; old use-sql-runtime + use-sql-store deleted. See "Outcome" below.

## Outcome (2026-07-01)

Final pushed structure (both remotes):

    701d8bdee  impl-runtime-store-5 (#1271)   [untouched]
     -> 6a008f64b  impl-secret-store (#1272)   reshaped: 10 secret + a68-PROD + psbt unit test
     -> 5b7bcee0d  impl-runtime-store-6        prep, 23 commits, default = KVDB
     -> a055ef3d7  wallet-default-sqlite       final, 2 commits: kvdb->sqlite flip (task 282) + a68 SQL test

- Route B + option A: config infra (runtime_store_config.go + open/create wrapper)
  lives in the prep defaulting to KVDB; the final flips the default to SQLite. No
  introduce-then-replace.
- a68baf9c0 split FINER than planned: production (signer.go + psbt_manager.go) AND
  the mock-based psbt unit test -> #1272; only the SQL-config signer_test.go -> final.
- Verified: per-commit build + test-compile 36/36; all 36 signed (G); final tip ==
  c0141c3f3 (content-preserving); full ./wallet test green at prep tip and final tip.
- #1272 CI not chased (per decision). impl-runtime-store-6 + wallet-default-sqlite
  have NO PRs yet (CI is pull_request-triggered) -> maintainer opens PRs, CI runs
  then; locally verified meanwhile.
- Scratch to prune: worktree .worktrees/routeb; branches impl-secret-store-reshaped,
  prep-tmp, f3-reconcile (+ ~250 stale backup/bkp branches).

## Goal

Reorganize the two top branches (`use-sql-runtime`, `use-sql-store`) so the
FINAL branch is the only place behavior flips (kvdb -> sqlite default).
Everything before it is **prep**: correct and tested on kvdb, with no dependency
on sqlite being the default. "Fix and prepare for the final shape, then flip."

## Key finding

The code ALREADY defaults to sqlite, and the flip is buried mid-stack in
`use-sql-runtime` (`runtime_store_config.go`, introduced by `wallet: add runtime
store config` + `wallet: select and open the runtime store`):

    // zero-valued Backend selects DBBackendSQLite, so new library-created
    // wallets default to the SQLite runtime store. Existing kvdb-only wallets
    // must set Backend to DBBackendKVDB explicitly to keep using walletdb.

~15 commits of fixes/features sit ON TOP of that default. The reorg delays the
flip to the very end: prep keeps **kvdb** as the default; the final branch adds
the config + flips to sqlite. The flip itself is Task 282.

## The 7 prep groups and their proper home

| Prep group (commits) | Belongs to | Why |
|---|---|---|
| 1. kvdb adapters: `add legacy wallet lifecycle helpers` / `add legacy signer derivation adapters` / `add legacy scan target resolver` / `assemble db.Store backend` | **impl-runtime-store series** | kvdb backend of the same `db.Store` interface the series built for SQL |
| 2. runtime-store wrapper (open kvdb; split out of `select and open the runtime store`) | **impl-runtime-store series** | wallet-side integration of the store; config-driven part is the only new piece |
| 3. route live paths: `route wallet lifecycle through kvdb` / `route signer pubkey lookups through kvdb` / `route syncer scan targets through kvdb` | **impl-runtime-store series** | identical in kind to the series' existing `route ... through Store` commits |
| 4. create/lifecycle correctness: `close Stop/Start race` / `prevalidate duplicate initial accounts` / `idempotent default account seeding` / `reject Create on existing` / `guard create-retry seed mismatch` (+ folded review fixes) | **impl-runtime-store series** | fixes to the store's CreateWallet/open/lifecycle flow |
| 5. imported-account fixes: `wallet+db: map imported account address schema` / `waddrmgr: key recovery scope types by account name` | **base-fix + impl-runtime-store** (NOT fix-import-schema — see "Prep 5 re-categorization" below) | bug sites pre-exist at the sql-wallet base / live in the syncer recovery keying; fix-import-schema only *exposes* them |
| 6. tx-conflict detection: `db/sql: add tx_inputs table and conflict-input queries` / `db: detect confirmed conflicts on shared external inputs` / `db/itest: cover confirmed conflicts` | **impl-runtime-store series** | extends the txstore's CreateTx conflict discovery |
| 7. `wallet: remove dead legacy DB helpers` | **general cleanup** | post-routing cleanup; belongs after the routing that orphans the helpers |

Stray review commit `review(import/sign): store-back deprecated imports, pubkey
derive, PSBT account` -> folds into the signer routing (impl-runtime-store) or
`impl-secret-store` if it touches keyvault.

## Takeaway

Almost none of this is new top-of-stack work:
- **impl-runtime-store series**: Prep 1, 2, 3, 4, 6, and the syncer half of Prep 5.
- **review-base-fixes (base bug)**: the `dbScopeAddrSchema` cast (Prep 5, commit 1).
- **general cleanup**: Prep 7.
- **genuinely new**: `runtime_store_config.go` + sqlite-default flip -> the final
  branch only (Task 282).
- **No fix-import-schema work** after verifying the code — Prep 5 was mis-labeled
  from subject lines; see "Prep 5 re-categorization" below.

## Prep 5 re-categorization (verified 2026-06-30)

Initial subject-line categorization put Prep 5 in fix-import-schema. Reading the
code shows neither commit belongs there.

**`033199dc3 map imported account address schema` -> review-base-fixes (BASE bug).**
`dbScopeAddrSchema` already does the raw cast
`db.AddressType(schema.ExternalAddrType)` at the sql-wallet base (90eaaccf2); the
`db.AddressType` enum is from the base too, and fix-import-schema's
`c3987f53e remove imported address scope` did NOT change the enum ordinals. The
mismatch (waddrmgr.PubKeyHash=0 vs db.RawPubKey=0) pre-exists the whole stack, so
there is no introducing commit -> new commit in the bottom `review-base-fixes`
branch (per the base-bug rule).

**`01996ab3c key recovery scope types by account name` -> impl-runtime-store (syncer).**
The `(scope, 0)` collision (imported-xpub masks to account number 0) manifests in
the recovery-scope keying in `wallet/syncer.go`, introduced by the
impl-runtime-store scan-horizon commits (`build Store scan batch params`,
`evict scan horizons on rollback`). The durable `Name` added to
`AccountScope`/`BranchScope` (base waddrmgr structs) folds alongside that syncer
fix. Not fix-import-schema.

**Exposed-by nuance.** fix-import-schema's `remove imported address scope` is what
routes imported-xpub accounts through the buggy cast and surfaces the masked-0
number — it *exposes* both latent issues without authoring the buggy lines.
Chosen (strict "where's the code"): base-fix + impl-runtime-store. Alternative
("exposed-by" = ownership): both fold into `c3987f53e remove imported address
scope`.

## Signer-layer split (verified 2026-06-30)

The signer work splits cleanly across two layers; only one piece is misplaced.

- **kvdb/legacy layer -> stays in runtime prep.** `afa1e7d58 kvdb: add legacy
  signer derivation adapters` + `f80c24761 route signer pubkey lookups through
  kvdb` wrap the legacy walletdb derivation (`DeriveManagedAddress` /
  `LoadManagedAddress` / `ResolveDerivedPrivKey`) and drop the direct `walletdb`
  import from `signer.go`. Same "route through kvdb" theme as the lifecycle/syncer
  prep (Prep 3). NOT impl-secret-store — this is the legacy plumbing, not the
  SQL-secret feature.
- **SQL-secret layer -> impl-secret-store (#1272, reshaped).** impl-secret-store
  already owns keyvault, `GetAccountSecret`, store-backed key derivation, and
  `route signer keys through Store on legacy miss`. The one misplaced piece is
  `a68baf9c0` (see review-fold above) -> fold into impl-secret-store's signer-key
  commit.

`recover SQL master seed on create-retry` (Prep 4) touches the master seed but is
wallet-create flow, not the runtime secret-read path -> stays create-path/runtime,
not impl-secret-store.

## Surgery required (not a pure reorder)

`wallet: select and open the runtime store` bundles the kvdb-opening wrapper WITH
the config + selection + sqlite-default. Split it:
- wrapper (opens kvdb directly) -> prep (Prep 2);
- `runtime_store_config.go` (DBBackend enum + SQLite/Postgres config structs) +
  config-driven selection + sqlite-default -> final branch.

The 3 create-path fixes that touch `runtime_store.go`
(`clean up partial wallet`, `reject Create on existing`, `guard create-retry
seed mismatch`) stay in prep, operating on the kvdb wrapper.

## Review-commit folds (per the fold-into-introducing-commit rule)

- `review: recover SQL master seed on create-retry; make Stop terminal` splits:
  seed-recovery -> `guard create-retry seed mismatch`; Stop-terminal ->
  `close Stop/Start lifecycle race`.
- `review: clean up partial wallet on post-Load failure; SQL birthday safety
  margin` -> the create/Load path commits.
- `review(import/sign): store-back deprecated imports, pubkey derive, PSBT
  account` (`a68baf9c0`) -> **impl-secret-store** (route B relaxed for #1272). It is
  SQL-secret/keyvault work (GetAccountSecret, `resolveDerivedPubKeyFromStore`
  matching the privkey fallback, PSBT account-secret resolution), mislabeled as a
  use-sql-store review commit. Fold into impl-secret-store's signer-key commit.

## Decision: Route B — continuation branches (chosen 2026-06-30)

Keep the green lower PRs (#1268, #1262-1271) **UNTOUCHED**; stack the reorganized
prep as honest continuation branches above `impl-secret-store`, with the sqlite
flip as the new top. No re-review/re-CI churn on the merged green PRs.

Rejected route A (fold each group down into its home branch): cleanest end-state,
but it rewrites green/open PRs and triggers full re-review + re-CI.

**Route B relaxed for #1272 (2026-06-30).** `impl-secret-store` IS reshaped — the
one exception to "keep open PRs untouched": it absorbs the misplaced signer/secret
review commit `a68baf9c0` so #1272 becomes the complete keyvault/secret branch.
Reshape + re-push #1272, but **its CI is NOT required to be fixed/green for this
reorg** (do not chase #1272 CI failures — leave to the maintainer). CI-green still
applies to the new continuation + `default-sqlite-runtime` branches; #1268-#1271
stay untouched. See "Signer-layer split" above.

### Layout (above the current tip)

    impl-runtime-store-5 (#1271, current tip)        [unchanged]
      -> impl-secret-store (#1272)                   [RESHAPED: absorbs a68baf9c0]
      -> <continuation branches for Prep 1-7>        [new, honest extensions]
      -> default-sqlite-runtime (FINAL: config + kvdb->sqlite flip; Task 282)

Continuation branches carry the prep groups, named as extensions of the series
they continue (e.g. `impl-runtime-store-6/7...`). The final `default-sqlite-runtime`
branch is the ONLY behavior flip. Exact branch count/grouping = next sub-decision.

### Base-fix consequence under route B

The `dbScopeAddrSchema` cast (Prep 5, commit 1) is a BASE bug. Route B does NOT
rewrite the base, so it cannot go in `review-base-fixes` (that would re-cascade the
whole green stack). Instead it rides in a continuation branch above the series
(fixed late, not at origin). Acceptable: it only manifests for imported-xpub
accounts, and these PRs are not the final merge target. Revisit if the maintainer
wants it folded to the base at final-merge time.

## Current state (do not lose)

**Route B is CHOSEN; the reorg is NOT yet executed and nothing is pushed.** The
SHAs below are the INPUT to the reorg, NOT push targets / NOT the route-B output.

`.worktrees/f3-reconcile` (branch `f3-reconcile` @ `c0141c3f3`) is a
content-preserving reconcile of the OLD final-3 onto the new lower tip. It still
carries the pre-reorg shape: `runtime_store_config.go` + the sqlite-default sit
mid-stack (in the use-sql-runtime commits), and `a68baf9c0` rides at the
use-sql-store level (after `f84e5d798`). The route-B OUTPUT (to be built) will:
move config + sqlite-default to the top (`default-sqlite-runtime`), reshape #1272
to ABSORB `a68baf9c0`, and regroup the rest into continuation branches.

- Lower stack round-3 GREEN on both remotes (stays untouched):
  base sql-wallet=90eaaccf2; fix-import-schema=8b5f6fc12 (#1268);
  prep-runtime-1=7755da1d9 (#1262); prep-runtime-2=e6e1f7f5a (#1263);
  prep-runtime-3=a0e618adf (#1269); impl-runtime-store=ba8aafd6b (#1256);
  -2=85d4d2640 (#1257); -3=c9e08a929 (#1258); -4=bbbf350b3 (#1270);
  -5=701d8bdee (#1271, TIP).
- f3-reconcile INPUT (old final-3, reconciled clean onto 701d8bdee; 34 commits,
  0 conflicts, per-commit build OK, signed) — boundary commits in that worktree:
  - impl-secret-store reconciled @ 7390b6514 (old 454b68cb1) — **does NOT include
    `a68baf9c0`; the #1272 reshape adds it.** So 7390b6514 is NOT the #1272 head.
  - use-sql-runtime reconciled @ f84e5d798 (old 2ec8f5f05)
  - use-sql-store reconciled @ c0141c3f3 (old b53ff9f7c) — `a68baf9c0` lives here,
    above f84e5d798.
- Next gate before executing: prep branch count/grouping.

## Cross-references

- Task 282: Default New Wallets To SQLite — the final flip (folds into the
  final branch here).
- Task 291: Rework fix-import-schema.
- Task 293: Additive normalized identity cutover plan.
