# Task 303 — Commit hygiene across #1272, #1291, #1292

Status: **DONE**, executed with Task 304 in one cascade and pushed. Heads
`7bf664f83` / `17638808c` / `d98e82bab`, all 13/13 CI green; the stack is now
**39 commits** (was 40; #29 folded into #27). One review follow-up landed on
top: `GetAddressSecretQuery.Validate` moved beside its struct, deleting
`addressstore_getsecret.go` (folded into #15).

Outcome: rule 1 PASS, rule 3 PASS on all 39, and rule-2 pairs **32 -> 18**. The
18 remaining are exactly the enumerated accepted debt (D2, D4, D5, the #27/#36
pairs) plus one regenerated-sqlc artifact (`GetAccountSecret` in #1 -> #14),
which is generated output rather than a hygiene violation. Record this count:
any future audit reporting more than 18 has found a regression.

Audit performed 2026-07-28 across the full stack
`sql-wallet..wallet-default-sqlite` — 40 commits at the time:

| PR | Branch | Commits | Head |
| --- | --- | --- | --- |
| #1272 | `impl-secret-store` | 1-19 | `7df7ca70c` |
| #1291 | `impl-runtime-store-6` | 20-38 | `f15eb8fe3` |
| #1292 | `wallet-default-sqlite` | 39-40 | `1f4bf7b52` |

Commit numbers throughout are positions in
`git rev-list --reverse sql-wallet..wallet-default-sqlite`.

The three rules audited:

1. Fold review fixes into the introducing commit; never add a commit to
   address a comment.
2. No introduce-then-remove / introduce-then-rename inside one series; build
   toward the end-state shape directly.
3. Every commit body explains the change and wraps at 72 columns.

## Audit Result

| Rule | Verdict |
| --- | --- |
| 1 — fold-in | **FAIL, 1 violation** (#29 fixes #27) |
| 2 — no introduce-then-remove/rename | **FAIL, 32 symbols in 16 pairs** |
| 3 — body content and 72-column wrap | PASS, all 40 commits |

Not every rule-2 finding is worth fixing. See **Tiering** below: the
recommendation fixes the #29→#27 pair (7 symbols) and the #1272 V1-V4 set, and
accepts the rest as documented debt — including V5's 2 signer helpers. 19 of the
32 live inside a single 2,736-line commit whose split would orphan five rounds
of review threads and force repeated rebases on #1300. Record the exact residual
count when execution finishes, per DoD item 2.

### Rule 3 — PASS

All 40 subjects are ≤50 characters, no body line exceeds 72 columns, no body
is empty, and each explains its change. Nothing to do.

### Rule 1 — one violation

**#29 `9b9b29dad wallet: fix shutdown teardown ordering + errors` is a fixup
for #27 `c270e8c0f wallet: remove migrated db ops`,** in the same PR:

- its subject begins with "fix";
- it touches 6 of the 7 files #27 touched (only `deprecated_test.go` is new);
- it deletes 5 helpers and 2 tests that #27 introduced — `requestStop`,
  `completeStop`, `finishStop`, `stopCompletion`, `waitForStopped`,
  `TestConcurrentStopReturnsCloseError`, `TestRequestStopNoOpWhenNotStarting`.

Folding it into #27 also removes 7 of the 32 rule-2 findings, which makes it
the highest value-per-risk change in this audit.

Otherwise rule 1 holds: no other commit exists to answer a review comment, and
none is a comment- or doc-only diff. Every review fix in rounds 1-5 landed
through `git commit --fixup` plus `rebase --autosquash`.

Commit 19 `wallet: use the default-account test constant` was the only other
candidate and was verified rather than taken on trust: the count of `"default"`
literals is identical at `sql-wallet` and at its own parent (3/15/4/8/6 across
the five test files, delta 0 everywhere). The stack introduces none of them, so
no commit here can absorb it. It stays standalone.

## Rule 2 — Where The Churn Is

No file is added then deleted anywhere in the stack. 32 *symbols* are built one
way and reshaped later, across 16 introduce → remove pairs:

| Symbols | Introduced by | Removed by |
| --- | --- | --- |
| 7 | #27 remove migrated db ops | #29 fix shutdown teardown |
| 4 | #3 pg: implement GetAccountSecret | #14 account secret adapters |
| 3 | #24 coordinate manager publication | #27 remove migrated db ops |
| 2 | #8 route signer keys on miss | #18 route signer through Store |
| 2 | #27 remove migrated db ops | #36 share SQL runtime stores |
| 2 | #27 remove migrated db ops | #35 persist create completion |
| 2 | #25 create through runtime stores | #36 share SQL runtime stores |
| 2 | #23 add runtime store open helpers | #27 remove migrated db ops |
| 1 | #7 add store-backed key primitives | #26 route sync runtime |
| 1 | #6 keyvault: wallet-secret genesis | #12 keyvault: passphrase lifecycle |
| 1 | #27 remove migrated db ops | #32 harden runtime sync state |
| 1 | #26 route sync runtime through Store | #27 remove migrated db ops |
| 1 | #24 coordinate manager publication | #30 serialize concurrent Load |
| 1 | #2 db: add account secret types | #14 account secret adapters |
| 1 | #17 route lifecycle through vault | #27 remove migrated db ops |
| 1 | #1 db: add GetAccountSecret query | #14 account secret adapters |

**#27 `c270e8c0f wallet: remove migrated db ops` is the epicenter** — it is a
party to 19 of the 32 findings: it deletes 7 symbols that #17/#23/#24/#26
introduced earlier in the same PR, and introduces 12 that #29/#32/#35/#36
delete later. It is also the largest commit in the stack at **2,736 weighted
added lines**, and its subject describes only the removal while the diff also
adds teardown machinery, sync helpers, and create scaffolding. Scope does not
match subject.

## Per-Commit Line Budget (advisory, not hard)

21 of 40 commits exceed 200 weighted added lines (production 100%, tests 50%,
generated 0%). Because this task explicitly permits #14, #18, #27 and #36 to
remain whole with a recorded justification, **200 is an advisory budget here,
not a hard cap** — calling it "hard" while allowing four exceptions was
internally inconsistent. A commit over budget must carry its reason in the
body; it is not required to split. The largest:

| Commit | Weighted |
| --- | --- |
| #27 `c270e8c0f` remove migrated db ops | 2,736 |
| #36 `574b68273` share SQL runtime stores | 2,202 |
| #38 `f15eb8fe3` coordinate shared chain resets | 1,758 |
| #26 `e7c9839a3` route sync runtime through Store | 1,687 |
| #18 `059c1f9ad` route signer through Store | 1,376 |
| #35 `194262b8c` persist create completion state | 956 |
| #29 `9b9b29dad` fix shutdown teardown | 830 |
| #11 `c8dda325a` waddrmgr: expose stored secrets | 825 |
| #14 `605d73cf9` account secret adapters | 793 |

The cap and this stack were never compatible, and the established practice
here is to justify size in the body ("This is forced-atomic…", as #14, #18 and
#26 already do) rather than to split. Enforcing the cap retroactively across
#1291 is a rewrite of the whole PR, not a touch-up.

## The Five #1272 Violations (detail)

### V1 — per-backend error mapper deleted after introduction

`mapGetAccountSecretErr` is added by #3 `2a3b91eb7 pg: implement
GetAccountSecret` and #4 `0e5ded279 sqlite: implement GetAccountSecret`, then
deleted from **both** backends by #14 `605d73cf9`, which replaces it with
`db.MapGetAccountSecretErr`.

### V2 — per-backend converter deleted and re-added generic

`accountSecretRowToInfo` is added non-generic by #3 and #4, then deleted by
#14 and re-added as `accountSecretRowToInfo[T accountSecretRow]` alongside the
new shared `db.AccountSecretRowToSecret`.

### V3 — `Validate` moved between files inside the PR

`GetAccountSecretQuery.Validate` is added by #2 `438b4e4d7` in
`wallet/internal/db/accountstore_getaccountsecret.go`, then moved by #14 to
`wallet/internal/db/data_types.go` with its receiver renamed `query` -> `q`.

**V1-V3 share one root cause.** The RF-001 deduplication from review round 4
landed in #14 instead of at the point of introduction. The shared `db` package
already exists at #2/#3, so shared helpers can live there from the start.

**But "introduce the final shape directly" is not literally achievable at
#2/#3, and the plan must say which strategy is used.** The final versions
depend on work that lands later:

- `GetAccountSecretQuery.AccountID`, added by #14;
- the generated `sqlc.GetAccountSecretByIdRow` types the final generic
  converter constrains on (`sqlite/accountstore_getaccountsecret.go:96`),
  which do not exist at #3/#4;
- three-selector validation, where #2 validates only Name/AccountNumber;
- the sentinel itself: #2 introduces `ErrInvalidAccountQuery` and #14 changes
  it to `ErrInvalidQuery` (so V3 is not a pure move — the behavior changes).

Two admissible strategies:

- **(a) Move AccountID earlier** — the SQL query, generated code, query field,
  and both backend implementations all shift into #1-#4. Yields the literal
  final shape, but drags generated output and SQL forward and inflates the
  early commits.
- **(b) Introduce durable two-selector helpers in #2, extended when AccountID
  lands.** `accountstore_common.go` lands with the types in #2, so #3 and #4
  consume it rather than declaring their own copies; #14 *extends* the helpers
  with the AccountID selector and the ById row type rather than deleting and
  re-adding them.

**Recommended: (b).** It removes the introduce-delete churn — which is what
rule 2 targets — while keeping #3/#4 standalone and leaving SQL and generated
output in their own commits, per the repo's existing practice. Under (b), #2
also carries the final sentinel (`ErrInvalidQuery`, moved from #14) and
AccountID stays a later extension. See the Execution Plan for the exact
per-commit shape.

### V4 — genesis test renamed within the series (narrow)

#6 `3e1317c84 keyvault: add wallet-secret genesis` introduces
`TestCreateWalletSecretsSpendableRequiresRoot`. #12 `7cf158ab5 keyvault:
complete passphrase lifecycle` renames and expands it into
`TestCreateWalletSecretsInputGuards`, a table test whose `"spendable nil root
key"` case preserves the original assertion.

So this is **test-name churn**, not lost coverage: introduce-then-rename inside
one series. The fix is for #6 to introduce the test under its final name and
shape, with #12 adding only the new cases.

**#6 asserts nothing about locked rotation**, so nothing is to be removed from
it on those grounds:

- `TestWalletVaultChangePassphraseLockedRequiresUnlockedState` is **base code**
  (`sql-wallet:wallet/internal/keyvault/vault_lifecycle_test.go:241`);
- #6 adds only `vault_genesis.go` and `vault_genesis_test.go`;
- #12 therefore changes *pre-existing base behavior* deliberately.

(An earlier version of this finding claimed otherwise, inferred rather than
verified; that claim is retracted.)

That behavior change is intended (Task 304's C1 keeps `OldPassphrase` precisely
so rotation works while locked). It needs a justification in #12's body, but it
is **not** a hygiene violation, and nothing should be removed from #6 for it.

### V5 — signer helpers deleted outright

#8 `2253c3dc3 wallet: route signer keys through Store on miss` adds
`(*Wallet).resolvePrivKey` and `(*Wallet).resolveDerivedPathPrivKey`. #18
`059c1f9ad` deletes both; neither exists at the tip.

#18's body claims forced atomicity for the *test* migration, which does not by
itself justify carrying two helpers the same PR removes.

**#8 cannot introduce the final helper set.** #18's signer depends on AccountID
lookup, address-secret queries, kvdb secret adapters, and imported-address
handling, all introduced between #8 and #18. So "just build the end state at
#8" is not available; the options are:

- **(a)** drop #8 and introduce signer routing once, after all Store
  prerequisites — which merges #8 into an #18 that is already 1,376 weighted
  lines;
- **(b)** reorder the prerequisite secret commits before signer routing — a
  reshuffle of most of #1272;
- **(c)** accept it as documented debt: #8 states in its body that its helpers
  are an intermediate step superseded by #18, and V5 joins the accepted-debt
  set.

**Recommended: (c).** (a) makes the worst line-cap offender worse, and (b)
reorders a PR that has absorbed five review rounds. Accepting it requires
DoD item 1 to be stated as "no *gratuitous* declared-then-removed symbol, with
the accepted set enumerated" — see the Definition Of Done, which was internally
inconsistent with leaving V5 open.

## Tiering — What To Actually Change

### Required

- **R1. Fold #29 into #27.** Rule-1 violation, same files, and it clears 7
  rule-2 findings at once.
- **R2. V1-V3** in #1272 — mechanical re-homing that removes churn.

### Required only because Task 304 already rewrites these commits

- **R3. V4** in #1272 — the genesis test rename. Doing it separately would
  conflict with 304's C1/C2, which edit the same commit (#12).

V5 is **not** in this tier: it moves to accepted debt (D4) because #8 cannot
build the final signer shape without reordering most of the PR.

### Considered and excluded

- **R4. #23 `d9c5bd945 wallet: add runtime store open helpers` and #24
  `0f00bdee6 wallet: coordinate manager publication`.** These introduce
  `openRuntimeStore`, `legacyKVDBConfig`, `beginCreate`, and `publish`, all four
  of which #27 deletes, so both commits read as dead ends. It is the most
  clearly *gratuitous* scaffolding in the stack.
  **Decision: excluded, accepted as debt (D5).** It is the same class as V5,
  it lives in #1291 which is already not being reshaped structurally, and
  including it means editing two more commits deep in the PR for no correctness
  gain. Not left conditional — the rebase plan assumes it is out.

### Accepted debt — do not reshape now

- **D1. Splitting #27 (2,736) and #36 (2,202).** These are the two structural
  problems and would normally be split. Reasons not to now: 19 findings sit
  inside one commit whose split is a multi-day operation across 19 dependent
  commits; it would orphan the inline review threads five rounds produced; and
  it forces repeated rebases on #1300.
  **Remedy: rename, then justify.** A body explaining that the subject is
  narrower than the diff does not fix a misleading subject — it documents it.
  #27 says "remove migrated db ops" while the diff also adds teardown, sync and
  create machinery, and absorbing #29 widens it further, so **rename it** to
  something representative such as `wallet: complete runtime store migration`.
  The body then explains the forced atomicity and the components. Same test for
  #36's subject.
- **D2. Single-symbol pairs** #7→#26, #17→#27, #26→#27, #24→#30. Several are
  legitimate behavior evolution across a 19-commit PR — e.g. #17's
  `TestControllerStop_LocksVaultWhenUnlocked` disappearing once #27 migrated
  the ops it asserted against. The rule targets gratuitous scaffolding, not
  every rename in a long series.
- **D3. The 200-line budget across #1291.** Justify in bodies rather than
  split.
- **D4. V5's signer helpers** (`resolvePrivKey`, `resolveDerivedPathPrivKey`).
  #18's signer depends on work landing between #8 and #18, so #8 cannot build
  the end state. #8's body must say the helpers are an intermediate step
  superseded by #18.
- **D5. R4's scaffolding** in #23/#24 — four helpers #27 deletes. Excluded by
  the decision above: no code changes. Both bodies do get a message-only line
  noting the helpers are superseded by #27, which is a rewrite step (see the
  Execution Plan) rather than debt recorded only in this file.

## Related

Task 304 (`304-keyvault-single-passphrase.md`) rewrites commits #10, #12, #13,
and #17 of this same stack. Both tasks cascade to #1291/#1292 and force a
rebase on #1300, so execute them in **one** pass rather than two. They also
collide inside #12, which 303 edits for V4's table cases and 304 edits for
C1/C2.

## Combined Rewrite Map (303 + 304)

Both tasks rewrite overlapping commits, and every fold shifts the positions
below it. **Build this map once, against the pre-rewrite numbering, before
touching anything** — otherwise the two plans apply to different histories.

| # | Commit | 303 | 304 |
| --- | --- | --- | --- |
| 2 | `438b4e4d7` db: account secret types | V3 + shared helpers + tests | — |
| 3 | `2a3b91eb7` pg: GetAccountSecret | V1/V2 strategy (b) | — |
| 4 | `0e5ded279` sqlite: GetAccountSecret | V1/V2 strategy (b) | — |
| 6 | `3e1317c84` keyvault: genesis | V4 final test name | — |
| 8 | `2253c3dc3` signer keys on miss | V5 body note (debt) | — |
| 10 | `40e55d573` waddrmgr: atomic passphrases | — | C5 fold + subject widen |
| 12 | `7cf158ab5` keyvault: passphrase lifecycle | V4 cases only | C1, C2 |
| 13 | `2abd0107b` kvdb: legacy vault shim | — | C3 sentinel |
| 14 | `605d73cf9` account secret adapters | AccountID extension only | — |
| 17 | `742cd735d` route lifecycle through vault | — | C4 + C7 |
| 18 | `059c1f9ad` route signer through Store | V5 stays as-is | — |
| 23 | `d9c5bd945` add runtime store open helpers | D5 body only | — |
| 24 | `0f00bdee6` coordinate manager publication | D5 body only | — |
| 27 | `c270e8c0f` remove migrated db ops | R1 absorbs #29; D1 rename+body | — |
| 29 | `9b9b29dad` fix shutdown teardown | **folded into #27** | — |
| 36 | `574b68273` share SQL runtime stores | D1 body | — |

C6's home is decided: the same widened #10 as C5, so **no base-fixes commit is
inserted** and no positions shift on its account. 304's C7 adds Controller work
to #17, which the table already reflects.

Order of operations: apply the lowest-numbered edits first and let the cascade
carry upward, re-deriving positions after each fold rather than trusting this
table past the first change.

## Execution Plan

**One workflow: a single chronological rebase from `sql-wallet`, editing each
commit in ascending order**, with one deliberate exception in the todo.

`#28 67b5af8bf itest: wire manager DB config to runtime store` sits **between**
#27 and #29, and a `fixup` line applies to whatever immediately precedes it. So
ascending order would fold #29 into #28. The todo must move #29 up explicitly:

```
edit   c270e8c0f  # 27 wallet: remove migrated db ops  (rename + body)
fixup  9b9b29dad  # 29 wallet: fix shutdown teardown ordering + errors
pick   67b5af8bf  # 28 itest: wire manager DB config to runtime store
```

Then continue chronologically. (Relying on `--autosquash` instead would require
rewriting #29's message to `fixup! <#27 subject>` first; reordering the todo is
the smaller operation. Either way, do **not** leave #29 in place expecting a
bare `fixup` to find #27.)

Steps, in the order the rebase reaches them:

1. **#2 — V3 + V1/V2 groundwork.** Introduce `Validate` in `data_types.go`
   with its final receiver name, and **move the `ErrInvalidQuery` declaration
   here** — it is currently declared by #14 and does not exist at #2's parent,
   so #2 will not compile without the move. Add `accountstore_common.go` with
   `AccountSecretRowToSecret` and `MapGetAccountSecretErr` in their **durable
   two-selector form**. Per strategy (b) these must *not* attempt the final
   generic row-fields shape — `sqlc.GetAccountSecretByIdRow` does not exist
   until #14.

   **Move the matching tests with the code.** #14 currently introduces
   `accountstore_common_test.go` (212 lines) and
   `accountstore_getaccountsecret_test.go` (83 lines); the two-selector
   validation and conversion cases move to #2 alongside the helpers, so #2 does
   not introduce substantive behavior untested. #14 then adds only the
   AccountID-specific cases. The backend and itest test files
   (`kvdb/accountstore_getaccountsecret_test.go`,
   `itest/accountstore_getaccountsecret_test.go`) stay where they are.
2. **#3, #4 — consume the shared helpers, declare no converter.** The exact
   shape, chosen so the symbol scan has one deterministic expected result:
   - do **not** introduce `accountSecretRowToInfo` in any form at #3/#4;
   - each backend builds `db.AccountSecretRow` from its own generated row inline
     and calls `db.AccountSecretRowToSecret` plus
     `db.MapGetAccountSecretErr` directly;
   - the generic adapter `accountSecretRowToInfo[T accountSecretRow]` is
     introduced **once, in #14**, when both generated row types exist.

   Rejected alternative: a one-member generic adapter at #3 whose constraint #14
   widens. It is also scan-clean, but a generic over a single type is noise, and
   this way the symbol appears exactly when it earns its generality.
3. **#6 — V4.** Give the genesis test its final name
   (`TestCreateWalletSecretsInputGuards`) and shape. Delete nothing on
   locked-rotation grounds; that claim was retracted.
4. **#8 — V5 (accepted debt).** Update the body to state that
   `resolvePrivKey`/`resolveDerivedPathPrivKey` are an intermediate step
   superseded by #18. **Leave #18 unchanged.** No code moves.
5. **#10 — 304 C5 + C6**, and widen the subject
   (`waddrmgr+wallet: rotate passphrases atomically`).
6. **#12 — 304 C1, C2**, plus V4's new table cases only.
7. **#13 — 304 C3** sentinel.
8. **#14 — extend** the shared helpers with the AccountID selector and the ById
   row type, rather than deleting and re-adding them, and add only the
   AccountID-specific test cases. Confirm #14 shrinks by roughly the deletions
   and moved tests it no longer carries.
9. **#17 — 304 C4 + C7.**
10. **#23, #24 — message-only edits** (D5). No code changes: R4 is excluded, so
    the helpers stay. Each body gains a line noting that its helpers are
    superseded by #27 — the same treatment #8 gets in step 4.
11. **#27 — absorb #29** via the `fixup` line, and rename to a representative
    subject (see D1). Rewrite the body for atomicity and component detail.
12. **#36 — D1 body** rewrite.
13. Re-run both audit scripts and confirm the remaining findings are exactly
    the accepted-debt set.

## Risks

- Rewriting #2/#3/#4/#6/#8 rewrites every commit above them, and folding
  #29 into #27 rewrites #29-#40. So #1291 and #1292 both cascade and need full
  re-verification and a force-push.
- **#1300 (`GustavoStingelin`, `itest: add Manager and Controller integration
  tests`) is based on `wallet-default-sqlite`.** Any rewrite forces another
  rebase on someone else's open PR. Coordinate before pushing.
- Folding #29 into #27 produces a ~3,500 weighted-line commit. That is worse
  on the line cap and better on rule 1 and rule 2. Accept it and say so in the
  body, or reconsider R1 — but do not leave a "fix" commit for the same PR.
- V1/V2 touch generated-adjacent SQL backends. Keep `.sql` and generated
  output in their own commits, and re-run `sqlc` for both dialects if any
  query text moves.

## Verification

Per rewritten commit:

- `GOWORK=off go vet ./...` plus an itest-tagged vet, since `go build` misses
  `_test.go` compilation.
- `git log --format='%h %G? %s' sql-wallet..wallet-default-sqlite` shows `G`
  on every line.

At each of the three PR heads:

- Four golangci passes: default, `./itest`, sqlite DB-itest, postgres
  DB-itest. **Pin the passes to SHAs, not branch names** — a branch-name run
  can race a fixup commit and lint the wrong tree.
- 80-column audit at **tab width 8** over added lines; `lll` runs at tab
  width 4 and cannot see that class.
- `git diff --check`.
- `make fmt-check` and `make sqlc-check` **mutate files before checking
  cleanliness**, so run them in a throwaway auxiliary worktree per head, never
  in the working checkout. All three stacked heads — and #1300 — must be
  checked independently, so that is four worktrees, discarded after use.

Branch level: `GOWORK=off go test ./wallet/...` and `-race` on the
concurrency tests.

## Definition Of Done

1. No commit in the stack exists to fix another commit in the same stack; #29
   is folded.
2. The symbol scan over `sql-wallet..wallet-default-sqlite` reports **no
   gratuitous** declared-then-removed symbol. The accepted set is enumerated
   here — D2's single-symbol pairs, V5's two signer helpers, and any #27/#36
   pairs left by choice — with its exact count recorded, so a later audit can
   distinguish intent from regression. Anything outside that set is a failure.
3. No file that this stack *introduces* is moved again later in the same
   stack. One-time relocation of pre-existing files is fine — #12's rename of
   `wallet/internal/bwtest/mock/vault.go` to `.../keyvaultmock/vault.go` is
   legitimate and must not be flagged.
4. No commit asserts behavior that a later commit in the same stack reverses.
5. #27 and #36 carry **representative subjects** — #27 renamed to something
   like `wallet: complete runtime store migration` — with bodies describing
   their atomicity and the components they cover. No commit relies on its body
   to excuse a subject that misdescribes the diff.
6. Rules 1 and 3 pass; commit 19 remains standalone with its justification.
7. Every rewritten commit compiles and passes vet standalone; all three PR
   heads are CI green with four clean lint passes.
