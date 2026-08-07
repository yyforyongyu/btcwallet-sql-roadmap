# Task 290: Finish SQL-review Phase 4 (quota re-split + use-sql-runtime test-clean + hygiene)

> **Archived 2026-07-01:** superseded by the task 294/295 reorg — the old use-sql-runtime stack was deleted/restructured; the quota re-split + test-clean items are moot.

Status: Open (deferred — to finish later).

## Background

The second SQL-review round (tasks 284–289 + structural 279/282/283) is
content-complete and verified green. All review fixes are landed and the
283 reshape is done. Only the Phase-4 *structural finishing* and the
*commit-hygiene* pass remain — all of it content-preserving (no behavior
change). NOT pushed.

## Current stack (13 branches on sql-wallet `4d7171ee9`, tip use-sql-store `7d9eaab3d`)

```
review-base-fixes    6f5abae24  +6
fix-import-schema    2d70c76ff  +14
prep-runtime-1       09c5a309e  +9
prep-runtime-2       3388a044f  +14
prep-runtime-3       56086b3a0  +7
impl-runtime-store   55aaf511f  +12
impl-runtime-store-2 d429f8bc4  +10
impl-runtime-store-3 9a5c8fdb9  +8
impl-runtime-store-4 73b77aed4  +10
impl-runtime-store-5 be78253c0  +15
impl-secret-store    dfd019c43  +10   (283: secret/key-material only)
use-sql-runtime      ce36deab7  +11   (283: legacy-kvdb runtime adapters + runtime SQL enablement)
use-sql-store        7d9eaab3d  +17   ⚠ OVER 15
```
`impl-secret-backend` retired (283). Empty-diff invariant holds vs the pre-283
tip `701adcbd7` (the 283 reorder was content-preserving). Verified at each step:
per-commit `go build`, `go vet`, `go test ./wallet ./wallet/internal/db/...`,
`gofmt -s`, `make sql` zero-diff, golangci `--new-from-rev=4d7171ee9` = 0.

## Remaining work (all content-preserving)

1. **Quota re-split** — `use-sql-store` is +17 (over the 15 cap). Peel the 3
   task-287 commits at the top into a NEW branch (e.g. `use-sql-conflicts`):
   - `b7c7c1ca8 db/sql: add tx_inputs table and conflict-input queries`
   - `b155ef3ac db: detect confirmed conflicts on shared external inputs`
   - `7d9eaab3d db/itest: cover confirmed conflicts on external inputs`
   Result: `use-sql-store` ≤15, new branch ≤15. 281-style boundary re-point.
   Verify `git diff 7d9eaab3d <new-tip>` is EMPTY (content-preserving).

2. **NotifyBlocks move (task 282 Item 2)** — extract the self-contained,
   dependency-free NotifyBlocks slice from `use-sql-store`'s
   `859608f97 review(syncer): restore btcd block ntfns + SQL sync fixes`
   (the `initChainSync` `s.cfg.Chain.NotifyBlocks()` hunk returning
   `"unable to start block notifications: %w"` + 3 `syncer_test.go` changes) and
   fold it into `use-sql-runtime`'s `ce36deab7 wallet+itest: support SQL wallet
   backends` (where `initChainSync`'s `NotifyReceived` already lives). Leave the
   SQL-coupled remainder of `859608f97` in use-sql-store. GOAL: `go test ./wallet`
   passes at the `use-sql-runtime` tip (`TestInitChainSyncWatchesUnspentForSpends`
   + `TestInitChainSync_NotifyBlocksError` go green there). Content-preserving for
   the final use-sql-store tree.

3. **Retitle stale commit subjects** — the 288 mirror-strip / folds left a few
   subjects overstating content; retitle to match. Known:
   - `a051f4c09 review(import/sign): store-back deprecated imports, pubkey derive,
     PSBT account` — the deprecated-import store-back was stripped (288); it now
     only does pubkey-derive + PSBT account. Audit the other `review(...)`
     commits for similar drift.

4. **Commit-hygiene audit** (skipped in the round, per maintainer) — run it per
   `feedback_commit_hygiene`: each commit minimal+necessary, scope matches title,
   no introduce-then-remove, weighted line caps (≤150 ideal / 200 hard;
   generated 0% / tests 50% / prod 100%), SQL+generated own commit, per-commit
   compile. (Note: SQL/cutover commits like `9c7f5f64b` and the 287/289 schema
   commits are sanctioned over-cap.)

5. **Scaffolding cleanup** — delete this run's temp/backup branches
   (`bkp-283-*`, `ckpt-*`, `rfx*`, any `bkp-rf-*`/`bkp-281-*` leftovers) and prune
   the agent worktrees under `.claude/worktrees/agent-*` once the above is green.

## Then (separate gate)

Push (confirm with maintainer first): both remotes, one branch at a time; the
NEW branch(es) need NEW PRs; existing branches update their PRs; coordinate with
Gustavo (the stack adopts+extends his keyvault scaffolding).

## Verification to re-run at the end
per-commit `go build`; `go vet`; `go test ./wallet ./wallet/internal/db/...`;
`make sql` zero-diff; sql-format + sql-lint; golangci `--new-from-rev=4d7171ee9`
(via `~/gc-cache`) = 0; all branches ≤15 + linear; legacy `deprecated.go`/`rpc/*`
untouched.
