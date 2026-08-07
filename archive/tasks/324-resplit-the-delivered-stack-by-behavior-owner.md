# Task 324 — re-split the delivered stack by behavior owner

> **CLOSED 2026-08-02 — archived.** Executed and promoted: #1272 `f56ea0241`
> (15 commits), #1291 `f647284c1` (6), #1292 `a2b629456` (1), on both remotes
> with every PR `headRefOid` matching. Boundary trees byte-identical to the
> inputs `c6425a509` / `df3a02daf` / `960dd02e8`, retained at
> `refs/backup/t324-pre/*`. Threads 0 unresolved; all three reviewers cleared
> the result. Only `Check commits` was pending at close, already verified
> locally across 23 revisions in both tag sets.
>
> `opus-report.md` is the surviving record; this file is evidence.

**Status:** **executed and promoted.** The re-split is delivered:

| PR | branch | head | commits |
| --- | --- | --- | ---: |
| #1272 | `impl-secret-store` | `f56ea0241` | 15 (was 7) |
| #1291 | `impl-runtime-store-6` | `f647284c1` | 6 (was 4) |
| #1292 | `wallet-default-sqlite` | `a2b629456` | 1 |

Pushed to `upstream` and `origin` under per-remote leases with full-OID source
refspecs; every PR `headRefOid` matches. Boundary trees byte-identical to the
inputs `c6425a509` / `df3a02daf` / `960dd02e8`, retained at
`refs/backup/t324-pre/*`. Thread sweep 112 / 135 / 34, **0 unresolved**. PR
bodies carry the real commit lists. Ledger and delivered→re-split mapping:
`.lgtm/pivot/t324-execution-ledger.md`.

**Open:** green CI at the delivered heads. Three commits exceed the 300 cap by
owner ruling (§2), each stating its measured overage in its own body. One
behavior-owner item is recorded open in `opus-report.md` §4: #1291 introduces a
sync-tip implementation it later replaces, inside the Manager commit that §5
records as unsplittable.

**Deviation from §7, recorded:** the PR-body edits went out immediately after
the push rather than after green CI. The bodies describe commit structure, not
results, so no CI outcome can falsify them — but the written order says
CI-then-metadata and that is not what happened.

Base pin `sql-wallet` = `2ea76dfc5e85acf84014a21e4813cb23945250fa`.

## 1. The one invariant

**The final tree at each PR head must be byte-identical to its input head.**

```bash
git diff --quiet <input-head> <rebuilt-head>   # must exit 0, all three
```

This is pure boundary reshaping. Because the trees are identical by
construction, every metric, gate and test result carries over unchanged and does
not need re-litigating: 15,269 / 5,803 / 9,322 / 105, inventory 7/43/43,
transaction boundary 23→18→0→0, both itests passing one `manager_create_wallet`
case. Only the *per-commit* and *stack-shape* gates are genuinely new.

**There is no sanctioned tree change.** If a split exposes a dependency defect
that cannot be resolved by moving existing hunks and their tests together,
**stop**: preserve the candidates, record the exact failing revision and its
falsifier, and defer the code fix to a new task. The owner's standing rule is
that existing bugs are fixed in a later PR. An executor may not change the tree
and merely note it — that would void every carried-over gate above.

## 2. Weight policy (owner, 2026-08-02)

**Four commits are exempt by owner ruling (2026-08-02).** After the rebuild,
`wallet: move the runtime store under the Manager` (1320.9), `wallet: route the
signer through the Store` (673.7) and `wallet: route chain sync through the
Store` (390.0) remain over cap. Each was attempted and each failed for a
recorded structural reason, not for convenience:

- the Manager's ownership fields, `kvdbBinding`, `defaultKVDBTimeout`,
  `storeFor`, `ownedStore`, `buildWallet` and `publish` are mutually recursive
  across four files, and the `NewManager` signature change requires every caller
  in the same revision. Config-alone, config-plus-binding and store-opener-alone
  were each built and each failed to compile;
- `signer.go`'s key-resolution chain is one interlocking rewrite. A three-way
  split was **built and measured**: 202.0 + 111.0 + 664.7 = 978 against 674 for
  the single commit, because a helper landed anywhere but its final position is
  re-added by its owning commit. Splitting inflated the change by 45%;
- `syncer.go` plus its test is 330.8 on its own;
- `wallet+db: add the account-secret store surface` (336.1) became the fourth
  when gpt found the original split published the Store contract with a kvdb
  adapter stubbed to `ErrAccountSecretUnavailable`. Removing that stub means
  folding the real 105-line kvdb implementation into the same commit, which
  costs 36 lines over the cap. A stub that compiles while its commit message
  claims three working adapters is the worse defect.

Owner ruling: **stop here.** "We don't want to over-complicate the split to
create more review churn." Further reduction would require hand-authoring
intermediate revisions of files that never existed in that state — commits that
compile but do not describe a real step — and §1 forbids the tree changes a
genuine behavioural split would need. Each of the three names its overage and
its reason in its own commit body.

**Otherwise: maximum 300 weighted added lines, no exception band, no cutover
escape.**
Every commit — and every manifest row in §4.3 — is at or below 300. If a change
appears not to fit without a temporary API or a tree change, **stop and request
a plan amendment**; the executor may not self-approve an oversized commit. The
escape an earlier draft allowed ("a cutover that cannot compile in pieces") is
withdrawn: it would have let the 1320.9-weight Manager commit declare its
`NewManager` cutover atomic and bypass this task's entire purpose.

| kind | weight |
| --- | ---: |
| hand-written production, SQL source | 100% |
| integration tests — any path segment `itest/` | 20% |
| ordinary `*_test.go` | 10% |
| generated (`*/sqlc/*`, `*.pb.go`) | 0% |
| deletions | 0% (never offset additions) |

```bash
git show --numstat --format= <oid> | awk '
  { a=$1; f=$3
    if (a == "-") next                                  # binary
    if (f ~ /\/sqlc\// || f ~ /\.pb\.go$/) next         # generated
    else if (f ~ /(^|\/)itest\//)  w += a * 0.20        # integration tests
    else if (f ~ /_test\.go$/)     w += a * 0.10        # unit tests
    else                           w += a               # production + .sql
  } END { printf "%.1f\n", w }'
```

The `(^|\/)itest\/` form is load-bearing: an earlier draft matched only
`internal/db/itest/` and missed the root `itest/` tree.

## 3. Disposition of the twelve input commits

Weights measured with §2's formula; independently reproduced by all three
reviewers.

| commit | weighted | disposition |
| --- | ---: | --- |
| `df3a02daf` Manager ownership | 1320.9 | **split** (§5.7) |
| `1beee395e` chain sync | 721.3 | **split** (§5.6) |
| `e553bb00c` signer routing | 689.6 | **split** (§5.5) |
| `dea193700` legacy vault | 429.2 | **split** (§5.4) |
| `99b4885d8` address-secret query | 395.2 | **split** (§5.2) |
| `a6c44e8f2` passphrase lifecycle | 336.9 | **split** (§5.3) |
| `0191cb119` account-secret query | 329.4 | **split** (§5.1) |
| `7b646f2c5` kvdb bridge | 169.7 | keep unchanged |
| `960dd02e8` itest backend | 137.8 | keep unchanged |
| `c6425a509` lifecycle→vault | 127.4 → 101.6 | **reallocate, then keep the lifecycle/Stop unit indivisible** |
| `a2b94236f` wallet-secret genesis | 124.3 | keep unchanged |
| `3cc760bf4` birthday margin | 7.0 | keep unchanged |

`c6425a509` is under the cap, but **"unchanged" was wrong and contradicted
§5.5.** `TestOpenWithRetryRoutedSigning` lives in this commit together with two
of the three `OpenWithRetry` repairs it needs — `kvdb.NewLegacyWalletVault` and
`Config{ChainParams}` — while the third, `newStoreRuntimeCache`, lives in
`e553bb00c`. Moving only the cache and the test would leave the earlier revision
red: the canonical SQL Vault cannot unlock kvdb secrets and the routed address
path has no chain parameters.

So the **complete** deprecated-constructor repair — the `wallet/deprecated.go`
wiring, the matching `wallet/manager.go` legacy-Load switch, and
`TestOpenWithRetryRoutedSigning` — moves to the first routed-signing commit in
§5.5. Measured: 25.8 weight leaves `c6425a509` (127.4 → 101.6) and joins
`e553bb00c` (689.6 → 715.4, which splits regardless): 8 production additions
in `wallet/deprecated.go`, 118 unit-test additions at 10%, and 6 production
additions in `wallet/manager.go`. The #1272 boundary tree is unaffected; only
hunk ownership changes.

What remains in the lifecycle commit stays **indivisible**: Vault routing, the
Stop-time lock, the strict mock expectations and the sole Stop-to-Vault
falsifier land together or the owning revision's own tests fail. That is the
§7.2 round-3 defect, where `Stop` changed in one commit and the expectations it
required arrived four commits later, leaving five commits red.

## 4. Execution protocol — do this before touching any delivered ref

**4.1 Back up the exact inputs.** `refs/backup/t316-pre/*` point at the *rev-6*
inputs and are not a restore point for this task.

Create each with **create-or-verify** semantics — create only when absent,
otherwise require it already equals the exact input OID. A bare `update-ref`
would silently overwrite diagnostic state on a retry:

```bash
# create only if absent (expected-old = zero); on failure, verify instead
git update-ref refs/backup/t324-pre/1272 c6425a509a4365822249c53e14433dace46389f8 \
    0000000000000000000000000000000000000000 \
  || test "$(git rev-parse refs/backup/t324-pre/1272)" = c6425a509a4365822249c53e14433dace46389f8
```

Repeat for 1291 `df3a02daf0d865265afef582f9b56c4d094f8334` and 1292
`960dd02e871987daeacf03b69b0eaef0b46b749b`. Verify all three read back before
proceeding.

**4.2 Rebuild on candidates only, in dependency order.** Create
`refs/heads/t324/candidate-{1272,1291,1292}` and work in dedicated worktrees
under `.worktrees/t324-*`. **Never rebuild on a delivered branch.** The
delivered refs keep their input OIDs until §4.5.

Create every candidate ref with expected-zero compare-and-swap. If one already
exists, do not reset, overwrite or silently reuse it: preserve its OID and
worktree, inspect whether it is the intended prior Task-324 attempt, and record
an explicit resume decision before continuing. Failed candidates are evidence,
not scratch refs.

Ancestry is **base → candidate-1272 → candidate-1291 → candidate-1292**, and
creating three candidates at their input heads does not establish it. Rewrite
candidate-1272's range over the base first; then rebuild candidate-1291's **own
range only** onto the final candidate-1272 head; then candidate-1292's own range
onto candidate-1291. Each boundary is checked with §6's `git diff --quiet`
before the next one starts.

**4.3 The commit manifest is WAIVED (owner, 2026-08-02).** All three reviewers
asked for a reviewed manifest before any rewrite; the owner's call is to execute
directly — "I don't think we need a manifest". The rebuilt commits themselves
are the reviewable artifact, and they are strictly better evidence than a
document describing them. Nothing else in this section is waived: backups (§4.1),
candidate isolation (§4.2), CAS promotion (§4.5), and every gate in §6 still
apply, and they are what actually catch a bad split. The accepted trade is that
an impossible split is discovered mid-rebuild rather than on paper — §4.2's
preserved candidates and §1's stop-and-defer rule cover that.

The paragraph below is retained as guidance for what each commit must satisfy.

**The manifest is not written and is not required.** The paragraph below records
what each commit must satisfy; §6 gates it against the actual rebuilt history
rather than against a document. Wherever a later gate says "manifest", read "the
rebuilt commit itself".

**Each commit must carry —**
§5 is an *ownership inventory*, not a verified sequence: several of its rows
cannot compile in the order listed. Before rewriting anything, write
`.lgtm/pivot/t324-commit-manifest.md` with one row per proposed commit —
commit class, subject, owned files and hunks, dependency predecessor, attached
tests, and estimated weight. Every row must be ≤300 and every row must name
what makes its revision compile. Classify each row as hand-written or
**SQL+generated**; a SQL commit carries its query source together with the sqlc
output regenerated from it, and never mixes with hand-written Go.

Every type, field, interface, query and file must first appear under its final
name and signature — no
temporary API that a descendant removes or renames. The manifest is reviewed
before the rewrite starts.

**4.4 Record every candidate OID and its measured weight** before promotion.

**4.5 Promote with compare-and-swap**, in dependency order, and
**worktree-aware**. A detached promoting worktree does not protect a delivered
branch checked out *somewhere else*: moving that ref leaves the other worktree's
index and tree describing the old commit while `HEAD` names the new one. This is
live — the repository root currently has `wallet-default-sqlite` checked out.

Before promoting: read `git worktree list --porcelain` and require that **no**
delivered branch is checked out in any worktree. A checkout is eligible to
detach only when its index and tracked worktree are clean. Inventory and
preserve untracked files—especially the untracked `.lgtm/` task evidence—and
verify no untracked path collides with the target tree. Stop on a staged or
tracked modification, or an unexplained untracked collision. Then, from a
tracked-clean detached worktree:

```bash
git update-ref refs/heads/<branch> <candidate-oid> <input-oid>
```

Accept only the expected input OID, or the candidate value if already promoted.
Abort on any third value — someone else moved the ref. After each swap, verify
the branch ref and that every worktree still satisfies the same tracked-clean,
untracked-inventoried condition before advancing.

**4.6 Roll back in reverse dependency order** with the same compare-and-swap
rule, restoring from `refs/backup/t324-pre/*`. Preserve failed candidates; never
overwrite a ref that a third party moved.

Local rebuild authorization is **not** push or PR-metadata authorization. §7
governs anything external.

## 5. Split map — ownership inventory

Ownership guidance for the manifest in §4.3, **not** a compiling sequence.
Adjacent hand-written rows merge freely up to the cap. A SQL+generated row does
not merge with a hand-written one. Where a row's
dependency problem is already known, it is named.

### 5.0 The intermediate-commit rule

**No behavior-changing production commit may be separated from the pre-existing
test adaptations that exact revision needs to pass.** These commits change
existing routing, so a production-only revision leaves the old tests asserting
the old behavior and lands red — the §7.2 failure class.

Explicitly allowed, because they change no behavior and compile on their own:
preparatory SQL+generated commits, final interface types, concrete methods
pre-landed before an interface widens, and genuinely supplemental test tables.
What is forbidden is an *untested behavior-changing* production commit.

An interface and its mocks must never arrive before every implementation they
require.

**SQL source and the sqlc output regenerated from it go in one commit together,
separate from hand-written Go.** This has now been ruled twice, the same way
both times, so it is not open. The owner's standing rule is explicit — "put a
`.sql` query/migration change together with the `.sql.go` it regenerates (run
`sqlc generate` in that same commit) in a dedicated commit; the hand-written
code that consumes the query goes in separate commits". A revision of this task
briefly mandated three exclusive classes — source alone, generated alone, then
adapters — on the reading that each deserves its own dedicated commit. That
reads the rule backwards, and gpt showed the concrete harm: a source-only
commit ships query source that does not match the checked-in sqlc output,
`go test ./...` cannot see it, and per-commit `sqlc-check` is delegated to
final-head CI (§6) — so **no gate would catch it**. Generated lines weigh zero,
so there is never a weight argument for separating them either.

A generated signature may not land while an existing caller still requires its
old form. Fold the final query shape backward, or reorder the hand-written
owner, so the pure SQL+generated commit compiles.

**The one admissible exception is a compile dependency (owner, 2026-08-02):**
"SQL edits should always be in one commit — unless there is a compile issue."
When a query that already has callers **at the pinned base** is re-keyed, the
generated params struct changes under those callers, and no ordering leaves them
compiling apart: nothing in this stack introduced them, so there is nothing to
fold backward. Those adapters — and only those — join the SQL+generated commit,
and the commit body must say which caller forced it.

This is not a general adapter escape. Convenience never qualifies; the test is
that every alternative ordering fails to compile. In practice it arose exactly
once: `db+sql: key the address-secret query on the script` re-keys
`addresses.sql` from `a.id` to `a.script_pub_key`, and the pg and sqlite
adapters both predate the stack. §5.1's account-secret family did **not** need
it — the correction folds backward instead.

**Test ownership respects PR boundaries.** Some tests covering #1272 behavior
were introduced in #1291. Moving them backward would change the #1272 tree and
violate §1. Attach each test to the earliest commit **within its own PR**.

### 5.1 `0191cb119` — account-secret query (329.4)

Introduce the **final** account-secret shape directly. Land the PostgreSQL and
SQLite query source together with the sqlc output regenerated from it, in one
SQL+generated commit; no account-secret adapter exists yet, so the new generated
method has no caller to break. Then land the data types, validation, Store
contract, mocks, three concrete adapters, wallet cache activation and shared DB
integration proof as hand-written ownership rows. The manifest may split those
hand-written rows further if their exact combined weight exceeds 300, but may
not fold hand-written Go into the SQL+generated commit.

### 5.2 `99b4885d8` — address-secret query (395.2)

waddrmgr secret accessors and cipher-provenance with their focused tests
(177.5). This commit's second-stage account-secret correction folds
**backward** into §5.1, so nothing here rewrites a shape §5.1 published. For the
address query, land the final PostgreSQL and SQLite source together with its
regenerated sqlc output in one SQL+generated commit, then the hand-written DB
types, Store contract, adapters and contract tests. No SQL address-secret
adapter exists before that hand-written row, so the generated signature can land
first without breaking a caller. Keep `ScriptIsSecret` with the hand-written
adapter change that first needs its provenance; never fold it into the
SQL+generated commit.

### 5.3 `a6c44e8f2` — passphrase lifecycle (336.9)

**Known dependency:** the final `Vault.ChangePassphrase` signature, the concrete
`WalletVault` method, the mock method, every existing caller and the lifecycle
tests are **one API cutover** — they cannot be three commits as an earlier draft
proposed. Moving or deleting the mock package before all import sites move also
breaks compilation. Expect a preparatory commit plus one cutover.

### 5.4 `dea193700` — legacy vault (429.2)

kvdb address-secret lookup with its essential tests (209.5); then
`LegacyWalletVault` translation with its tests. Keep the core address-secret
behavior with its falsifiers — only genuinely supplemental table cases may
follow separately. No new kvdb format or migration.

### 5.5 `e553bb00c` — signer routing (689.6)

Address import provenance and Store writes; `WalletVault` cipher selection
including the retained `CKTPublic` mapping; account public-key lookup and child
derivation; private-key/script resolution and signing; PSBT routing.
**Known dependency:** routed signer methods read `w.cache`, so the
`OpenWithRetry` cache repair and `TestOpenWithRetryRoutedSigning` must land with
the *first* routed signing path, not with a later lifecycle owner; and most PSBT
expectation rewrites are forced by signer routing and cannot wait for a later
PSBT row. Add no new real-backend PSBT scenario — that deferral stands
(`opus-report.md` §5.5).

### 5.6 `1beee395e` — chain sync (721.3)

Store data/adapters for sync and recovery; account/address manager routing;
recovery target resolution and stable Store-ID stamping; syncer transaction,
rollback and watch-state routing; tx-reader/utxo/signer fallout.
**Known dependency:** `db_ops_test.go` contains Manager Create/Load tests whose
production subjects stay live until §5.7 — only the sync/recovery-owned
deletions move here; the Manager-owned ones wait. No standalone mass-deletion
commit. No live-watch redesign, reset coordinator, or public `TxStatus`.

### 5.7 `df3a02daf` — Manager ownership (1320.9)

The largest and most coupled split; the manifest **must** carry a dependency
table naming the exact symbols and callers per commit. "Land each helper with
its first caller" is not sufficient guidance.

Ownership groups: `ManagerConfig` and backend validation; runtime Store
configuration/open helpers; kvdb binding ownership; Manager construction and
`Close` ordering; `Create` through the selected Store; `Load`/build/cache
publication; caller migration in `bwtest` and the existing itest wiring.

**Known dependencies**, which the manifest must solve symbol by symbol:
`defaultKVDBTimeout`, `kvdbBinding`, the Manager ownership fields, `storeFor`,
`ownedStore`, `buildWallet`, `publish`, and the public `NewManager` caller
cutover — which requires every wallet, benchmark, `bwtest` and itest caller in
the same commit. `Create` calls `buildWallet` and `publish`.

An earlier draft claimed that pre-landing the three files' 17 unexported
functions would trip `unused`. **That was wrong** — Go permits unused
package-level functions, this task runs no per-commit golangci-lint (§6), and
GitHub's per-commit job only compiles. Pre-landing a final helper is a valid
split whenever it compiles and changes no behavior.

Delete `wallet/db_ops.go` where its Store replacements are already active.
Preserve one `NewManager`, one config per component, Manager-owned database
lifetime, one in-process kvdb wallet, serialized caller assumptions. Do not
restore `DBConfig`, `OpenStoreParams`, store handles, create manifests, write
gates, atomic account creation, or lifecycle races.

## 6. Gates

**Per rebuilt commit:**

- `GOWORK=off go test ./...`, using **one dedicated scratch `GOCACHE` created
  once for the whole loop** and removed afterwards — this machine's shared cache
  drops entries under checkout churn and produces phantom import failures. Do
  not wipe the cache per commit; that is ~24 cold builds for no gain. The gate
  is: **every touched package passes**, and the only tolerated broad-run
  failure is an environmental package failure reproduced at the pinned base in
  the same isolated environment. Reproduce that class once at base and record
  it. Do **not** require exact subtest-set equality: prior runs here produced
  different failing `BitcoindEvents` subtests across unchanged revisions, and
  sometimes the package passed outright. Tolerate the `./chain` bitcoind class,
  nothing else.
- `git diff --check <oid>^ <oid>` — the patch, not the worktree.
- present-tense subsystem-prefixed subject ≤50; explanatory body wrapped at
  about 72 columns; no `Co-Authored-By: Claude` trailer; `%G? = G`.
- the manifest class matches the patch: a SQL+generated commit contains only
  `.sql` source and the sqlc output regenerated from it — no hand-written Go —
  and every hand-written commit contains neither.
- §2's weight, recorded.

**Per rebuilt PR boundary:**

- `git diff --quiet <input-head> <rebuilt-head>` — this is a boundary gate; an
  intermediate commit is not expected to equal a PR head.

**Per rebuilt stack:**

- the introduce-then-remove audit from `opus-report.md` §5.4, re-derived over
  the **new** commit list — the removed-name set is unchanged, the commits it
  sweeps are not;
- a manifest-to-history final-shape audit covering types, fields, interfaces,
  query names and files in addition to functions: no item is introduced and
  later removed or renamed within a rebuilt PR range;
- base → #1272 → #1291 → #1292 ancestry and the three PR scopes unchanged.

These are history-specific validations, not a local CI battery. **Do not run
local CI-equivalent gates** — `gofmt -l`, `go vet`, `go install`, tagged itest
compilation, or the Docker-backed `lint-check`/`fmt-check`/`sqlc-check`. GitHub
CI is their authority (`opus-report.md` §3.3), and review rounds do not run them
either.

## 7. Promotion

Nothing external moves without authorization, per operation class, naming every
remote. When authorized, follow the protocol archived at §316-12.2: one branch
at a time in #1272, #1291, #1292 order; full-OID **candidate** source refspec
with a branch-and-remote-specific `--force-with-lease`; verify each ref, then
the PR `headRefOid`; never plain `--force`; never push from a mutable branch
name.

**Partial-publication recovery, activated with Task-324 OIDs** (the archived
§316-13 mechanics, made concrete rather than left implicit). Before the first
push, obtain and record: pre-authorized per-remote rollback tuples; reverse
dependency order; candidate-OID destination leases; the captured **old full
OIDs** as immutable push sources; and stop-on-any-third-party-value. A rollback
that cannot name its exact tuple is not authorized.

For PR bodies, restore a snapshot **only** when the live body still equals the
exact text Task 324 wrote — a concurrent human edit must never be overwritten.
Replies and thread resolutions are irreversible; record them as such after any
partial failure rather than attempting to undo them.

Require green CI at the rebuilt OIDs **before** any metadata mutation, and
re-check the affected PR head immediately before each body edit, reply and
resolution. Replies and thread resolutions are irreversible — record them as
such.

Metadata owed at promotion:

1. **PR body commit lists** — all three enumerate the commits this task
   replaces. Snapshot Task-324 old/new bodies separately under
   `.lgtm/pivot/pr-bodies/`.
2. **#1291's "why the last commit is atomic" paragraph** must go once that
   commit is split.
3. `.lgtm/pivot/t315-commit-manifest.md` gains a delivered-OID → rebuilt-OID
   map, old mapping labelled historical.
4. Re-run the fully paginated thread sweep; require zero unresolved.

The two known-false body sentences were corrected in the `.body.new` snapshots
on 2026-08-02 but **not yet in the live bodies** — #1272 claimed
`wallet/deprecated*` is byte-identical to base (`deprecated.go` is +9/−5 and
`deprecated_test.go` adds 118 lines), and #1291 claimed a `w.cfg.DB` → `w.db`
substitution that was deliberately reverted. Push those corrections with the
commit-list update.

## 8. Out of scope

No behavior change to make a split easier. No feature, itest case, PostgreSQL
Manager goal, migration, atomic-create work, lifecycle hardening, watch-only
redesign, or live-watch work. No review-fix commits stacked on top — every
correction folds into its owning commit and the history is re-signed. Deferrals
keep their owners: #317 atomic create, #318 duplicate create and kvdb name
identity, #319 lifecycle races, #320 ADR 0012 and watch-only, #321 PostgreSQL
Manager, #322 shared-chain reset, #323 the kvdb→SQL migration.

## 9. Definition of done

1. All three rebuilt trees byte-identical to the input heads (§1).
2. Every commit ≤300 weighted added lines by §2's formula, except the four
   named in §2, each stating its measured overage and structural reason in its
   own commit body. No other commit may exceed the cap.
3. Every commit satisfies section 6's test rule: every touched package passes,
   and the only tolerated broad-run failure is the base-reproduced, untouched
   `chain` BitcoindEvents environmental class without exact subtest-set
   equality.
4. Both introduce-then-remove audits in §6 clean over the new commit list.
5. Every commit is either hand-written Go or SQL+generated (`.sql` source with
   the sqlc output regenerated from it), never both; all commits satisfy §6's
   message and signature rules; ancestry and PR scopes are preserved; backups
   and candidate refs are recorded per §4.
6. Authorized push, green CI at the rebuilt OIDs, zero unresolved threads, and
   every §7 metadata item done.
