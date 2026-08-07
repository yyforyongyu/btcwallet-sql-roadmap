# Task 305 — #1272 passphrase corrections

Status: **NON-EXECUTABLE — historical evidence, superseded by [[Task 313]].**

> Do not execute this file. Its leading plan **restores the dual-passphrase API
> that [[Task 313]] deletes** (F0), and Task 308, which it says it executes
> through, is itself archived. F0, T7, F9's public half and F5's rotation tests
> are all reversed. Only F1's accepted-request completion and F6's commit hygiene
> survive, and 313 carries them forward explicitly.

## Scope, After The Library Directive

`wallet/` is a library and its API is the component interfaces;
`wallet/deprecated.go` is not part of it. That removes most of what this task
originally carried, because much of it existed to keep legacy and kvdb
passphrase shapes working.

**Kept — all of it is new-API work or reduces code:**

- **F0** — revert C1/C2/C3/C7. Restores the two-passphrase `keyvault` API and
  `LegacyManagerVault` delegation. A revert, so it *removes* code.
- **F1** — `Controller.ChangePassphrase` accepted-request completion. The same
  defect class as F2 but on the interface we actually ship.
- **F5** — the red `-race` test: two parallel tests share a package-level
  `rootKey`. Kept because a red test blocks the merge, not because the code it
  covers is in scope.
- **F6** — commit hygiene: #10's subject widened to `waddrmgr+wallet:`, helper
  tests moved.
- **T7** — revert the exported `waddrmgr.AddrStore` break. Also a revert.
- **F9** — public `wallet.Err*` sentinels, since `keyvault` is internal. Matches
  the `utxo_manager.go:36-49` convention.

**Dropped — deprecated-only, recorded as future work:**

- **F2** — the `Wallet.ChangePassphrases` guard bugs. The code is already merged
  in #1272, and these are two real defects (it blocks forever on a never-started
  wallet, and can report failure after the locker already accepted) — but in a
  method outside the library API. **Not fixed here, and no code `TODO`** — a
  marker in a deprecated method would open a tracking surface for exactly the
  code this round declared out of scope. Threads `Ue9wZ` and `UfRnj` carry the
  record instead.
- **F3** — dropped as a gate. Retargeting it at `Controller.ChangePassphrase`
  over a Manager-loaded kvdb fixture would still make a kvdb fixture acceptance
  work for this round. If the coverage is wanted later it can be added on the
  supported API without a kvdb wallet.
- **F4** — dropped as a gate: the partial-zero it fixes is in the legacy
  rotation path (`changePassphrasesStaged.commit`), not the SQL vault. Real, and
  recorded.
- **F7** — dropped entirely. Its bolt-reopen mechanics are not a deliverable and
  nothing else needs the fixture now.
- **F8** — only the docs the kept items touch; the deprecated-surface sweep is
  dropped.

Everything below is the original analysis. Where it prescribes
deprecated-surface work, the dispositions above govern.

## F0 — The Design Conclusion: Revert, Don't Patch

**Task 304's C1/C2/C3/C7 were wrong and must be reverted.** The pre-existing API
was already correct; narrowing it removed a capability the transition needs, and
everything built afterwards was scaffolding to recover it.

What the original shape handled, and the narrowed one does not:

| Case | Original behavior |
| --- | --- |
| SQL wallet, public requested | `WalletVault` rejected it |
| kvdb, lnd (pub == priv) | delegated both to waddrmgr atomically |
| kvdb, standalone btcwallet (pub fixed `"public"`) | caller states each half |
| kvdb, `--noseedbackup` (two different old values) | caller states each half |
| kvdb loaded through `Manager` | worked — the vault holds `db` + `mgr` |

The mistake was reasoning "SQL has one passphrase, so the canonical API should
have one." The same API must serve kvdb wallets for the whole window between lnd
adopting the new API and each wallet being migrated. The two passphrases cannot
be derived from one value, because **we cannot assume pub == priv**: lnd sets
both to the user's password, while btcwallet's own daemon keeps the public one
at its configured default (`walletsetup.go:184`, `config.go:271`). A single
fixed policy breaks one of them either way:

- always rotate both -> breaks standalone btcwallet, whose config keeps passing
  `"public"` at open;
- never rotate public -> breaks lnd, which opens with the user's password as the
  public passphrase.

### F0 revert plan

| Task 304 change | Fate |
| --- | --- |
| C1 narrow `keyvault.ChangePassphraseParams` | **revert** |
| C2 delete `WalletVault`'s public rejection | **revert** |
| C3 `LegacyManagerVault` refuses | **revert** — restore delegation |
| C7 Controller handler rejection/mapping/no-op | **revert** |
| C4 `isKVDB` selection comment | keep; corrected in #1291's replay |
| C5a legacy handler retargeted at the atomic API | keep — see F4 |
| C5b `ChangePassphrases` added to exported `AddrStore` | revert — see T7 |
| C6 guard on `Wallet.ChangePassphrases` | keep, fix per F2 |
| kvdb format invariant test | keep, replace per F7 |

Restore both tests deleted in that round:
`TestLegacyManagerVaultChangePassphrasePrivate` — the test that would have
caught this — and `TestWalletVaultChangePassphrasePublicUnsupported`.

The restored API permits **public-only, private-only, or both**: standalone
btcwallet uses the singular methods, lnd rotates both. So the contract is *not*
"both must be stated"; it is that **each requested half supplies its own
independent old and new values**. Document it that way.

`ChangePassphraseRequest` needs no field change — it is base code and already
correct — but gains:

```go
// A canonical (SQL) wallet is guarded by a single passphrase, so only the
// private fields apply; setting ChangePublic returns
// ErrPublicPassphraseUnsupported. A legacy-format (KVDB) wallet has two
// independent passphrases: each half you request must supply its own old and
// new values, which cannot be derived from the other half.
//
// TODO: the public fields exist only for the legacy format and are removed
// with KVDB support after the KVDB-to-SQL migration.
```

## Review Findings

### F1 — High: the Controller API has the same ambiguous-success bug

`sendReq` accepts the request, then `waitForResp` can return `ctx.Err()` or
`ErrWalletShuttingDown` while the handler continues and commits
(`controller.go:807-815`, `:1328-1340`). A caller is told the rotation failed
while the new passphrases are live — the same defect as F2, on the new API.

**Fix.** Once a passphrase request is accepted, wait for the real result.
`waitForResp` is shared by every controller operation, so do **not** change it
globally; give the passphrase path its own wait that does not abandon an
accepted request. Regression tests for cancellation and for shutdown, both
asserting the caller learns the true outcome.

**Pin the invariants the fix rests on.** Waiting on `resp` alone after
acceptance is deadlock-free only because of two properties that are true today
and must be asserted so a refactor cannot silently break the fix:

1. **`requestChan` is unbuffered** (`manager.go:1490`: `make(chan any)`), so
   acceptance is a rendezvous — the main loop has already taken the request and
   runs the handler synchronously before it can observe shutdown; and
2. **every handler path writes `req.resp`** (`controller.go:1267-1312`) and
   `resp` is **buffered** (`make(resultChan, 1)` at `:1159`, `:1166`, `:1175`),
   so the reply is never lost and the handler never blocks delivering it.

If either changes — a buffered `requestChan` would permit accepted-but-never-
processed requests at shutdown — the dedicated wait additionally needs a
loop-exited escape plus a final non-blocking `resp` read to report the true
outcome. State both invariants in the code and assert them in the tests.

The F2 side needs the same treatment, where the invariants also already hold:
`changePassphrases` is unbuffered (`deprecated.go:7246`), `walletLocker` replies
on every path (`:2716-2752`), and `errChan` is buffered (`:2887`). F2's "no
quit escape after accept" is correct **because** the locker always responds
before stopping; name that invariant so implementation and tests preserve it.

**Document the new cancellation boundary on the exported method.** This narrows
`ChangePassphrase`'s contract: the context still aborts *before* `sendReq`
accepts, but an accepted credential mutation is no longer abandoned — the call
waits for its definitive result even after the context is done. That is a
deliberate trade (an ambiguous success is worse than a late return) and must be
stated in the exported comment, or the next reader will "fix" it back.

The cancellation test must therefore **cancel after an explicit acceptance
latch**, not merely after calling: cancelling before acceptance exercises the
pre-`sendReq` path and would pass against the buggy code too — the same vacuous
-barrier shape this stack has rejected repeatedly.

**This is a per-path fix over a systemic defect, and that must be recorded.**
Accept-then-abandon exists for **every** operation routed through
`sendReq`/`waitForResp`, not just passphrase rotation. Fixing only the
passphrase path leaves the new API internally inconsistent and lets the special
case fossilize.

It is still the right fix *here*: passphrase rotation is the one operation where
an ambiguous success means the caller loses access to a wallet whose credentials
did change, and `waitForResp` is shared by every operation, so changing it
globally in a passphrase commit would be an unreviewable blast radius.

The systemic defect is real and stays recorded: once the handler loop accepts a
request, its outcome should be reported regardless of caller cancellation, for
**all** operations — not only this path. It is **not assigned**. Task 10002 was
created for it and is now retired as a duplicate of ground the read-only
[[Task 1000]] reference already describes, so cite neither as an owner. Note the
scope of the exception in the #1272 commit body and leave the general fix
unscheduled.

### F2 — High: never-started blocks, and accepted-then-abandoned

Two defects in `Wallet.ChangePassphrases`:

1. It sends on an unbuffered channel while `quit` is still open and no locker
   goroutine exists, so the send blocks forever; the guard only covers a nil
   `walletDeprecated`. The "never started" test closed `quit`, making it
   byte-identical to the "stopped" test — a vacuous test of the family this
   stack has been rejecting in review all round.
2. After a successful send, `walletLocker` owns the request and its `walletdb`
   transaction may commit, yet the second select can still return
   `ErrLegacyWalletUnavailable`.

**Fix.** Snapshot `started`, `quit` and `changePassphrases` **once** under
`quitMu`. The rationale applies to **`quit` alone**: `StartDeprecated` replaces
`w.quit` (`deprecated.go:1535-1540`), so reading it twice observes two
generations. The request channels are made once at construction (`:7245-7246`)
and never replaced, so snapshotting them is harmless consistency, not a fix —
say so, rather than implying they also rotate.

Refuse deterministically when not running. After a successful send, wait on
`errChan` with no quit escape. Tests: never-started with `quit` **open**,
stopped, shutdown-after-accept, and the snapshot race.

### F3 — High: the prescribed lnd verification cannot run — waive it explicitly

**Persist this rationale in #1272's commit body**, not only here — a waived
verification whose reasoning lives in a planning file reads later as a
verification simply not done.

An external `go.work` build of lnd fails before any test runs: btcwallet is on
btcd v0.26.0 while lnd still imports legacy `btcd/chaincfg`, `btcd/wire` and
`btcd/txscript`. I prescribed this check without executing it.

**Decision: waive the downstream build, and substitute in-repo coverage.** The
DoD must say plainly that **full downstream lnd compilation is unverified** and
why. Replace it with btcwallet tests covering both lnd password shapes:

Both must run through the **concrete `Wallet.ChangePassphrases`** method, not
`waddrmgr.Manager.ChangePassphrases` directly: only the exported path covers
lnd's actual call site, the wallet locker, and the C5/F2 integration.

- equal public/private (lnd's main path), rotating both to one new value;
- unequal public/private (`--noseedbackup`: `"public"` / `"hello"`), rotating
  each half with its own old value.

Each case must then verify reopen and unlock behavior with the new values.

### F4 — High: rotation must do full lock cleanup, not a partial zero

The commit path zeroes only `masterKeyPriv` and the passphrase hash
(`waddrmgr/manager.go:1394-1398`) rather than the full lock cleanup, so a
concurrent unlock can leave decrypted material resident after rotation marks the
manager locked.

**Decision: mandate the full `m.lock()`-equivalent cleanup.** Documentation-only
narrowing is not acceptable for an exported primitive that advertises defensive
concurrency safety. Tests must assert the crypto keys, account xprvs, cached
addresses, and scripts are all cleared.

### F5 — Medium: the required race verification is red

`GOWORK=off go test -race ./waddrmgr -run
'TestChangePassphrasesConcurrent(Lock|Unlock)$' -count=1` races: both parallel
tests mutate the shared test `rootKey` through `setupManager`.

**Fix.** Give each test its own root key. A test defect, not a production race,
but the check is in the DoD and must be green.

### F6 — Medium: Task 303 steps documented but not applied

Confirmed against the pushed heads:

- `7577a4709` (#2) introduces the shared account-secret helpers **without their
  tests**; they remain in `eaa734ea6` (#14). The plan said to move them.
- `eb9bf3192` (#10) keeps the narrow `waddrmgr:` subject although it also
  changes `wallet/deprecated.go`; widen it to `waddrmgr+wallet: rotate
  passphrases atomically`.
- `a2cfce510`, `46093ae27`, `d8b56f7db` bodies describe the removed delegation
  model. Mostly resolved by the F0 revert; re-check each against final code.
- `ErrPassphraseChangeUnsupported` lands unused in `a2cfce510`. Moot after F0,
  which removes it.

### F7 — High: keep a real Manager-loaded kvdb rotation test

The `Manager` and `LegacyManagerVault` wiring already exists at the #1272 head,
so this is testable now and must not be deferred. Without it the F0
revert is asserted only through a mock vault returning nil, which is what let
the original dead end ship.

Create a fixture through the deprecated `Loader`, close it, load it through
`Manager`, rotate through `Controller`, reopen with the new **public**
passphrase, unlock with the new **private** passphrase, and assert the old
values are rejected.

### F8 — Medium: docs that become false after the revert

- The F0 note must say "each requested half supplies its own old and new
  values", not "both must be stated" — the API allows public-only and
  private-only.
- `wallet/manager.go:1467-1468` (the C4 comment) says rotation "stays on
  `Wallet.ChangePassphrases`", which the revert makes false. **That correction
  belongs to #1291's replay** ([[Task 308]] cascade step 4), because C4 landed
  in #27 — a #1291 commit, not #1272. Do not touch it here.

Then audit the remaining stale comments: `controller.go:103-105` and
`:1264-1266`, `kvdb/vault.go:113-117`, `keyvault/vault_lifecycle.go:218-226`,
and the orphaned doc block left in keyvault's `var` group.

### T7 — Decision: use a local extension interface

C5 added `ChangePassphrases` to the exported `waddrmgr.AddrStore`, breaking
every external implementation at compile time.

**Decision: revert that and use a local extension interface plus a type
assertion at the call site.** A failed assertion must return a stable error, not
panic, with a negative test covering an `AddrStore` that does not implement it.

**Record its expiry.** The assertion exists only while kvdb support does. Extend
the kvdb-removal TODO to name the extension interface *and* its negative-test
path, so the workaround is deleted with kvdb rather than outliving its reason.
This overrides the earlier preference against
narrow subset interfaces: that preference concerned internal store subsets, and
does not justify a breaking change to an exported interface for one caller.
Record the reasoning in #10's body.

### F9 — Sentinels must be reachable by external callers

`keyvault` lives under `wallet/internal/`, so
`keyvault.ErrPublicPassphraseUnsupported` **cannot be imported** by external
users of `Controller`. Re-export or translate it as
`wallet.ErrPublicPassphraseUnsupported`, and cover it with `errors.Is`.

While there, expose `wallet.ErrInvalidPassphrase` too — that addresses the
stable-error finding raised on #1300.

## GitHub Threads

**This task owns #1272's seven.** Verified counts: #1272 = 7, #1291 = 5,
#1292 = 1. [[Task 308]]'s DoD 4 dispositions all thirteen, including the newer
`UgPWI`, `UgRON` and `UgVTm`.

| Thread | File | Planned fix |
| --- | --- | --- |
| `Ue1m1` | kvdb/vault.go:130 | F0 revert — redirect dead end disappears |
| `Ue2Iz` | kvdb/vault.go:113 | F0 revert restores described behavior |
| `Ue9uZ` | kvdb/vault.go:130 | F0 revert; plus its stale-doc list (F8) |
| `Ue9wZ` | deprecated.go:2900 | F2 — one snapshot under `quitMu` |
| `UfRnj` | deprecated.go:2910 | F2 — no quit escape after accept |
| `UfRnp` | controller.go:791 | F6 — subject prefix and bodies |
| `UfRn0` | waddrmgr/interface.go:157 | T7 — local extension interface |

### Anchor mapping: this task rewrites #1272, but some anchors are stack-tip

Several line references above are positions at `d98e82bab` (the #1292 head)
because that is where the reviews read them. The folds work at the
`impl-secret-store` head, where the same code sits at:

| Symbol | Cited (stack tip) | At #1272 head |
| --- | --- | --- |
| `ChangePassphrase` entry | `controller.go:807-815` | `:421-439` |
| `sendReq` | — | `:830-843` |
| `waitForResp` | `:1328-1340` | `:845-856` |
| F4 partial zero | `waddrmgr/manager.go:1394-1398` | `:1387-1391` |
| `requestChan: make(chan any)` | `manager.go:1490` | `manager.go:349` |
| `resp: make(resultChan, 1)` | `controller.go:1159/66/75` | `:682/689/698` |
| `handleChangePassphraseReq` | `controller.go:1267-1312` | `:782` |
| `sendReq` | `:1315-1327` | `:830` |

The `deprecated.go` anchors in F1's and F2's invariant paragraphs
(`:1535-1540`, `:7245-7247`, `:2716-2752`, `:2887`) were verified correct **at
#1272** and need no mapping; only the `controller.go`/`manager.go` ones drift.

The controller comment anchors (`:103-105`, `:1264-1266`) and the C4 anchor
(`wallet/manager.go:1467-1468`) are correctly HEAD positions, since that comment
is corrected in #1291's replay. **Re-verify every line reference at execution
time** —
they drift with each fold.

## Commit Ownership And Rewrite Order

**#1272 commits only.** Subjects, not SHAs. Apply in ascending order in one
rebase from `sql-wallet`; re-derive positions after each fold.

| # | Subject | Work |
| --- | --- | --- |
| 2 | `db: add account secret types` | F6 move helper tests here |
| 10 | `waddrmgr+wallet: rotate passphrases atomically` | T7, F6 |
| 11 | `waddrmgr: expose stored secrets` | F5 root isolation |
| 12 | `keyvault: complete passphrase lifecycle` | F0 revert C1/C2; sentinels |
| 13 | `kvdb: add legacy manager vault shim` | F0 revert C3, restore test |
| 14 | `wallet+db: add account secret adapters` | F6 tests move out |
| 17 | `wallet: route lifecycle through vault` | F0 C7, F1, F9 aliases |

**F2 is not in this map.** It was assigned to #10; the rescope drops it. The
`ChangePassphrases` guard already merged in #1272 keeps its two defects; the
record is the answer on threads `Ue9wZ`/`UfRnj`, with no code marker.

**F3, F4 and F7 are not in this map either** — all three are legacy-path work.
F4's partial zero and F2's two defects are real and recorded; neither gates this
round.

## Push, Cascade And Threads

**This task does not push #1272 alone.** Rewriting from commit #2 changes every
descendant SHA, so #1291 and #1292 must be replayed onto the new head in the
same operation. That cascade is owned by **[[Task 308]]'s Execution Cascade** —
an earlier version of this section handed it to Task 306, which is retired.

Threads: the seven on #1272 are dispositioned in 308's DoD 4 —
`Ue1m1`/`Ue2Iz`/`Ue9uZ` by F0, `Ue9wZ`/`UfRnj` answered as dropped-F2,
`UfRnp` by F6, `UfRn0` by T7. The five on #1291 and one on #1292 are also
308's; an earlier line here claimed Task 306 owned six descendant threads.

**#1300 is not deprioritised** — it is the immediate next dependent, and it
receives the final #1292 SHA from 308's cascade step 7.

## Verification

At the final #1272 head: `go vet ./...` per commit (it compiles `_test.go`),
the four golangci passes pinned to that SHA, an 80-column audit over lines added
relative to the backup ref, and `git diff --check sql-wallet..<final-1272>`.
Run `make fmt-check` and `make sqlc-check` in a throwaway worktree.

## Definition Of Done

Scoped to #1272, rescoped for the three-PR round.

1. **F0 reverted**: two-passphrase `keyvault` API restored, `WalletVault`
   rejects the public half, `LegacyManagerVault` delegates, and both deleted
   tests are recovered from `7cf158ab5`
   (`wallet/internal/keyvault/vault_lifecycle_test.go`) and `2abd0107b`
   (`wallet/internal/db/kvdb/vault_test.go`).
2. **F1**: an accepted `Controller.ChangePassphrase` request returns its
   definitive result. The `requestChan`-unbuffered and buffered-`resp`
   invariants are asserted, and the exported comment states the cancellation
   boundary. The cancellation test cancels **after an explicit acceptance
   latch**.
3. **F5**: the shared package-level `rootKey` no longer races between parallel
   tests; the suite passes under `-race`. This one is kept despite living in
   legacy test code, because a red `-race` test blocks the PR from merging.
4. **F6**: #10's subject is `waddrmgr+wallet:`, helper tests sit with the code
   they cover, and every commit body describes its own change within 72
   columns.
5. **T7**: `ChangePassphrases` is off the exported `waddrmgr.AddrStore`; the
   call site uses a local extension interface with a stable error, not a panic,
   and a negative test covers a store that does not implement it.
6. **F9**: the passphrase sentinels are nameable as `wallet.Err*` by an
   external package, following the `utxo_manager.go:36-49` convention.
7. The dropped items are recorded, not silently gone: their threads are answered
   "valid finding, deferred from these three PRs, not yet assigned". **No code
   `TODO` for F2** — marking a deprecated method would open a tracking surface
   for code this round declared out of scope.

---

# Archived Analysis (non-executable)

Everything below predates the rescope and the three-PR round. It is retained
because its findings are verified against the code, **not** because it is a
plan. Where it prescribes deprecated-surface work, a `sql-wallet` cascade, or
Task 306 ownership, the sections above govern.

## Commit Ownership And Rewrite Order

**#1272 commits only.** Subjects, not SHAs. Apply in ascending order in one
rebase from `sql-wallet`; re-derive positions after each fold.

| # | Subject | Work |
| --- | --- | --- |
| 2 | `db: add account secret types` | F6 move helper tests here |
| 10 | `waddrmgr+wallet: rotate passphrases atomically` | F2/F3/T7/F6/F4 |
| 11 | `waddrmgr: expose stored secrets` | F4 asserts, F5 root isolation |
| 12 | `keyvault: complete passphrase lifecycle` | F0 revert C1/C2; sentinels |
| 13 | `kvdb: add legacy manager vault shim` | F0 revert C3, restore test |
| 14 | `wallet+db: add account secret adapters` | F6 tests move out |
| 17 | `wallet: route lifecycle through vault` | F0 C7, F1, F7, F8, aliases |

**F4 splits across #10 and #11; F5's test isolation is #11's.** The production
cleanup belongs in **#10**, where `changePassphrasesStaged.commit` — the code
with the partial zero — is introduced; leaving the fix in #11 would ship a
commit that knowingly under-zeroes. Its assertions belong in **#11**, because
`waddrmgr/manager_secret_test.go` is introduced there and putting them in #10
would mean introducing the file early or duplicating it. F5's shared-`rootKey`
isolation is likewise #11's. Decided, not optional.

**#10's subject is `waddrmgr+wallet: rotate passphrases atomically`.** F6 found
that commit carrying `wallet/` behavior under a `waddrmgr`-only subject and body
(thread `UfRnp`), so the widened subject is the fix — the map must not keep
showing the old one.

**Sentinel ownership splits across two commits.** #12 owns the internal
`keyvault` sentinels; #17 owns the public
`wallet.ErrPublicPassphraseUnsupported` and `wallet.ErrInvalidPassphrase`
aliases plus the Controller-facing tests.

### Push and validate #1272 only

Push `impl-secret-store` alone, with `--force-with-lease`. Do **not** require
#1291 and #1292 green: this task is scoped to #1272 and rewriting from #2 makes
its descendants diverge by construction.

**Record the divergence:** after this push, `impl-runtime-store-6` and
`wallet-default-sqlite` no longer descend from the new `impl-secret-store` head.
Task 306 rebases and rewrites them; until then #1291 and #1292 are stale on
purpose, not by accident.

## #1300 — Known Consequence, Deprioritised

Rewriting from #2 invalidates #1300's base again. It is already conflicting, 38
commits behind, with changes requested and 25 unresolved threads.

**The owner has deprioritised it for this round**, so this task does no work on
it. Recorded so the consequence is not a surprise: after the final push, #1300
needs a rebase onto the new #1292 head, a range-diff to confirm its four
authored commits survive intact, and fresh SQLite/KVDB CI. Reviewers have
flagged that leaving it is a risk; that call sits with the owner.

## Verification

**Scoped to #1272.** Verify the final `impl-secret-store` SHA and the range
`sql-wallet..<final-1272>` — not `sql-wallet..wallet-default-sqlite`, and not
the descendant heads, which this task knowingly leaves divergent.

Per rewritten commit: `GOWORK=off go vet ./...` plus an itest-tagged vet, and
`%G? = G`.

At the final #1272 head: four golangci passes (default, `./itest`, sqlite
DB-itest, postgres DB-itest) **pinned to that SHA**; an 80-column audit at **tab
width 8** over lines added relative to `refs/backup/t305/impl-secret-store`.
The repo's own standard is tab width 4 (`.golangci.yml:96` `tab-width: 4`,
`.gemini/styleguide.md:12`), so auditing at 8 is deliberately **stricter**, not
a contradiction: leading tabs count double, so passing at 8 implies passing
at 4.
It is a cheap over-approximation that leaves headroom for one more indentation
level.

Then `git diff --check sql-wallet..<final-1272>`. Run `make fmt-check` and
`make sqlc-check` in a throwaway worktree, never in the working checkout.

Branch level: `GOWORK=off go test ./wallet/... ./waddrmgr/...`, plus `-race` on
the concurrency tests including F5's.

Known-unrelated failure, confirmed identical at the pre-task baseline: the
`./chain` bitcoind tests (environment). The #27 itest-tagged vet exception does
**not** apply here — that commit is in #1291, so it is Task 306's concern.

### Test lifecycle precision

Two traps that will silently produce vacuous passes:

- **`Loader` creation does not start `walletLocker`.** Any test expecting
  `Wallet.ChangePassphrases` to *succeed* must reopen and start the legacy
  wallet first; otherwise it exercises F2's refusal path while appearing to test
  rotation.
- **The F7 Manager-loaded test must start the wallet** before rotating, and the
  bolt handle must be closed before reopening — otherwise it either rotates
  against an unstarted wallet or deadlocks on bbolt's exclusive file lock.

  **Correction to an earlier draft of this bullet, which was wrong twice over.**
  It cited `Wallet.Stop` at `controller.go:459` reaching `finishStop` ->
  `evictAndCloseStore`, and told the test to assert cache eviction and a store
  close. Verified at `7bf664f83`: `Stop` is at `:317`, and **`finishStop`,
  `evictAndCloseStore` and `runtimeStoreRegistry` do not exist at that head** —
  all three are #1291 work. `Stop` there only cancels the context, waits for
  goroutines, locks the vault and transitions state; `delete(m.wallets` and
  `.Close()` have zero hits across `manager.go`/`controller.go`/`wallet.go`.
  There is no Manager cache to evict and no owned runtime store to close, so
  those assertions are unwritable. (Also: `Manager.Close` genuinely does not
  exist, but the `Controller` interface **does** declare `Stop`
  (`controller.go:122`, `:146`) — the earlier "no `Controller.Stop` in this
  tree" was half-false.)

  **The real #1272 surface makes this easy.** `Config.DB` is `walletdb.DB` at
  that head (`wallet.go:173`, non-nil-checked at `:217`), and `Manager.Load`
  builds the wallet from the caller's handle —
  `kvdb.NewStore(cfg.DB, txMgr, addrMgr)` (`manager.go:325`) and
  `kvdb.NewLegacyManagerVault(cfg.DB, addrMgr)` (`:347`). `openLegacyStore` does
  not exist yet either. **The Manager never opens or owns the bolt file — the
  test does.** So: create the database, pass the handle in `Config.DB`, load
  through `Manager`, `Start`, rotate, `Stop`, then close the handle the test
  itself opened and reopen it. No `Manager.Close`, no `evictAndCloseStore`, no
  `(*kvdb.Store).DB` and no close-on-`Stop` change is needed, and there is no
  lock collision as long as the test closes what it opened.

  Note for [[Task 306]]: this confirms deleting `(*kvdb.Store).DB` is safe —
  F7's test does not need it.

## Definition Of Done

Scoped to #1272. Task 306 owns everything below it.

1. F0 reverted: two-passphrase API restored, `WalletVault` rejects the public
   half, `LegacyManagerVault` delegates, both deleted tests restored. **This is
   mechanical, not reconstructive** — the deleted tests survive in orphaned
   commits: `TestWalletVaultChangePassphrasePublicUnsupported` in `7cf158ab5`
   (`wallet/internal/keyvault/vault_lifecycle_test.go`) and
   `TestLegacyManagerVaultChangePassphrasePrivate` in `2abd0107b`
   (`wallet/internal/db/kvdb/vault_test.go`). Recover them from there.
2. F7's Manager-loaded kvdb rotation test passes, owned by #17: `Loader`
   fixture, closed; loaded through `Manager` with the `Controller` started;
   rotated through `Controller`; reopened with the new public passphrase and
   unlocked with the new private one; old values rejected — no mock vault.
3. F1 and F2 fixed, with the tests each can actually have.
   `Wallet.ChangePassphrases` takes **no context**, so F2 gets never-started
   (quit open), stopped, restart/snapshot race, and shutdown-after-accept.
   F1 — the Controller path, which does take a context — gets caller
   cancellation and shutdown-after-accept. Both must prove the caller learns the
   true outcome.
4. F3's waiver is explicit, and both password shapes are exercised through the
   concrete `Wallet.ChangePassphrases`. The DoD states that full downstream lnd
   compilation is unverified, and why.
5. F4 performs full lock cleanup, asserting crypto keys, account xprvs, cached
   addresses and scripts are cleared. F5 is green.
6. F6's four items applied. F8's comments match the code. T7 uses a local
   extension interface returning a stable error on assertion failure, with a
   negative test. F9's sentinels are reachable as `wallet.Err*`, owned by #17.
7. #1272's **seven** threads replied to and resolved, verified by a paginated
   sweep at the final `impl-secret-store` head.
8. `impl-secret-store` alone is pushed and CI green, with four clean lint
   passes at that head. #1291 and #1292 are knowingly divergent until Task 306
   rebases them; this is recorded, not treated as a regression.
