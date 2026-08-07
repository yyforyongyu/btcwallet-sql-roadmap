# Task 312 — review followups on #1272 / #1291 (post-push round)

Status: **NON-EXECUTABLE — thread evidence only, superseded by [[Task 313]].**

> Do not apply this file's disposition table as a patch list. §A is moot (the
> machinery containing the bug is never rebuilt), §B is fixed by 313's ownership
> change rather than a bolted-on guard, §C/§D are moot, §G is superseded, and
> §H1/§H2 become never-write rules. 313's re-disposition table governs. What
> remains valuable here: the verified findings and the thread inventory
> (6 / 2 / 0, with #1291's two on page 2).

Source: the 8 unresolved review threads on the three PRs as of 2026-07-30, after
the [[Task 308]] round pushed `impl-secret-store` `0d98a7d45`,
`impl-runtime-store-6` `8b4f29f74` and `wallet-default-sqlite` `1243c274c` (all
CI-green). Split: **6 on #1272, 2 on #1291, 0 on #1292.** Section H adds two
deletions carried over from the *previous* round, reopened by owner correction.

Every claim below was checked against the code at those heads. Two are confirmed
defects, several are cheap and correct, one has an inaccurate premise with a
valid underlying concern, one is already tracked, and one should be pushed back
on.

> Pagination note: #1291 has `totalCount=121`, so a single
> `reviewThreads(first:100)` query reports **zero** unresolved and both of its
> findings are invisible. Page 2 is required. Same trap as the previous round.

## A. `U1T6T` — locked-state wipe is incomplete (merge-blocking)

`waddrmgr/manager.go:1387`, in `changePassphrasesStaged.commit`. Reviewer
(`gpt-5`) is **correct**.

The re-assert branch does only:

```go
	if s.lockedAtStage || m.IsLocked() {
		m.masterKeyPriv.Zero()
		zero.Bytea64(&m.hashedPrivPassphrase)
		m.locked.Store(true)
	}
```

`(*Manager).lock()` additionally zeroes `cryptoKeyScript`, `cryptoKeyPriv`,
every `acctInfo.acctKeyPriv`, and every cached address secret through
`wipeAddressSecret` (including `lastExternalAddr` / `lastInternalAddr` warmed by
`loadAccountInfo`), and it ends with `m.locked.Store(true)` of its own.

So after **stage-while-locked → concurrent `Unlock` → commit**, the manager
reports `IsLocked() == true` while cleartext private key material stays
resident. That is the invariant locking exists to enforce, and the path is
exactly the one `TestChangePassphrasesConcurrentLock` /
`TestChangePassphrasesConcurrentUnlock` already exercise — not theoretical.

**Fix: call `m.lock()` instead of the partial wipe.** It does everything the
branch does plus the rest, and the locking discipline already matches — `lock()`
does not take `m.mtx` itself; its callers hold it (`Close()` at `:510` acquires
at `:496`; `Lock()` at `:1768` acquires at `:1760`), and `commit` holds `m.mtx`
for its whole body. No new helper needed.

Then extend the race test to assert `cryptoKeyPriv`, `cryptoKeyScript`,
`acctKeyPriv` and the cached address secrets are cleared, not just that the
manager reports locked — the current assertions pass with secrets resident,
which is why this survived review.

Owning commit: `waddrmgr+wallet: rotate passphrases atomically` (#1272).

## B. `U1VsY` + `U1V9E` — `Manager.Load` has no mutual exclusion

Both on #1291; **one fix**, so do them together. Reviewer (`gpt-5`) is correct
on both.

**`U1VsY` (`wallet/manager.go:1053`).** `Load` calls `awaitCreate` (`:1048`),
builds, then `publish` (`:1060`) — and never calls `beginCreate`. `Create` does
(`:772`) and holds its `release` until `publish` at `:934`. So two concurrent
`Load`s for one name both miss the cache, both open independent store handles,
and both publish; the overwritten `*Wallet` stays live with an open pool and is
no longer tracked by the manager.

**`U1V9E` (`wallet/controller.go:665`).** Teardown evicts via `w.onStopped()`
*before* `closeRuntimeStore()`. The existing comment justifies that ordering —
it stops a concurrent `Load` from being handed an instance whose stores are
closing. The reviewer identifies the half the comment omits: in the window
between evict and close, a fresh `Load` misses the cache and opens a **second
pool on the same file** while this instance still owns the first, violating the
one-pool-per-file constraint. Both concerns are real and in tension; evict-first
alone cannot satisfy both.

**Fix.** Serialize with the per-name construction guard that already exists —
`publish` (`:1197`), `beginCreate` (`:1220`), `awaitCreate` (`:1268`):

- take the guard in `Load` around build-and-publish, mirroring `Create`;
- hold it across evict + close in teardown so publication cannot resume until
  the close completes;
- if two racers still build, close the losing handle rather than leaking it.

**These are the limitations this round documented rather than fixed** — #1291's
PR body items 2 and 4, and the `TODO(yy)` markers on `Manager.Load`
(`wallet/manager.go`) and `openRuntimeStore` (`wallet/runtime_store.go`). Fixing
them retires two of the six code TODOs and two PR-body entries.

## C. `U1H_s` — document that `ScriptIsSecret` is inert on SQL

`wallet/internal/db/data_types.go:877`. Add the explanation plus a `TODO(yy)` to
remove the field once kvdb support is dropped.

Correct and cheap. Consistent with [[Task 309]] candidate 6 and with
[[Task 311]], which is what makes the removal possible. Keep the existing doc
paragraphs that explain *why* kvdb needs it — the addition is the SQL-inertness
statement and the removal marker, not a replacement.

## D. `U1Wts` — mark the unconditional legacy vault as temporary

`wallet/manager.go:347` (#1272 head) builds the wallet with
`keyVault: kvdb.NewLegacyManagerVault(cfg.DB, addrMgr)` unconditionally. #1291
replaces that with the `isKVDB` branch that selects between the legacy adapter
and `keyvault.NewWalletVault`.

So "temporary, fixed in the following PR" is accurate. Add the `TODO` naming the
follow-up so a reviewer of #1272 alone does not read the hardcoded legacy vault
as the intended end state.

## E. `U1UGB` — premise is wrong, the churn is real

`wallet/signer.go:597`. The thread says commits `48342446e` and `eff0a24c5`
"introduce and then remove" `resolvePrivKey` and `resolveDerivedPathPrivKey`.

**They are not introduced in this stack.** Both exist at `sql-wallet`
(`signer.go:804` and `:829`). `wallet: route signer keys through Store on miss`
*modifies* their signatures (threading `ctx`), and `wallet: route signer through
Store` *deletes* both. So this is modify-then-delete, not
introduce-then-delete — do not adopt the framing verbatim when replying.

The waste is still real and lands in the same category as the standing
no-introduce-then-remove rule: the earlier commit spends a signature change on
functions the later one removes.

**Before folding, check independent value.** `route signer keys through Store on
miss` does more than touch these two helpers — it routes key resolution through
the Store on a cache miss. If that carries standalone meaning, the right fix may
be to leave the signature churn out of it rather than to collapse the two
commits.

## F. `U1Vqn` — already tracked

`wallet/deprecated.go:2686` — "why are we touching deprecated files here". Owner
directive, recorded in full with the inventory and fix in [[Task 310]]. Reply
should point there rather than re-litigate.

## G. `U06ol` — push back

`wallet/internal/keyvault/keyvault.go:40`: "do we really need `ChangePublic` or
`ChangePrivate`? can't we just infer it by checking whether `PublicOld` is empty
or not?"

**Inference is unsafe, because an empty public passphrase is legal.** waddrmgr
rejects only the *private* passphrase when empty —
`if !isWatchingOnly && len(privPassphrase) == 0` (`manager.go:2434`), and
`ErrEmptyPassphrase`'s own doc says "the private passphrase was refused due to
being empty". There is no check anywhere rejecting an empty `pubPassphrase`. So
under the proposed inference, a wallet whose public passphrase is empty could
never request a public rotation: empty would mean both "not requested" and
"currently empty".

**And the field pair is already scheduled for deletion.** [[Task 311]] makes
single encryption the invariant for every backend, which removes the public half
of `ChangePassphraseParams` entirely. Redesigning the struct now to infer a flag
that is about to disappear is churn.

Recommended reply: the empty-passphrase evidence, plus the pointer to 311.

## H. Deletions reopened by owner correction

**The principle, corrected.** "Minimal" means a **smaller, more reviewable
diff** — not "these branches must not change further". Deleting unnecessary
surface *serves* that goal. The previous round answered `UgPWI` and `UgRON` as
"valid finding, accepted and deferred from these three PRs, not yet assigned" on
the mistaken reading that touching the branches again was itself the thing to
avoid. That reasoning was wrong; both are straightforward deletions that shrink
#1291.

Both threads are currently **resolved** on GitHub, so acting on them needs a
fresh reply on each rather than a silent change.

### H1. Delete `kvdb.IsMissingStore` (`wallet/internal/db/kvdb/driver.go:35`)

Introduced by `kvdb: add runtime store lifecycle adapters` (#1291, first
commit). The whole body is one line:

```go
func IsMissingStore(err error) bool {
	return errors.Is(err, walletdb.ErrDbDoesNotExist)
}
```

Exactly **one** caller: `wallet/manager.go:1435`
(`case kvdb.IsMissingStore(err):`), introduced by `wallet: complete runtime
store migration` in the same PR. The reviewer's "why do we need this thin
method?" is well founded.

**Do not naively inline it.** `wallet/manager.go` imports `walletdb` **zero**
times at #1291, and keeping it that way is the point of this whole series;
inlining `errors.Is(err, walletdb.ErrDbDoesNotExist)` would re-introduce that
import into `wallet/` and work against the static `walletdb` boundary check
#1291 plans.

**Correct shape: replace the predicate with a sentinel.** Map
`walletdb.ErrDbDoesNotExist` to a package-level sentinel where the kvdb store
produces the error, and have the caller use `errors.Is(err, <sentinel>)`.
Preferably map onto an existing `wallet/internal/db` sentinel so `wallet/`
compares against the `db` contract rather than against kvdb at all. Net effect:
one exported function fewer, the import boundary preserved, and the call site
reads as an ordinary sentinel comparison.

### H2. Delete `(*kvdb.Store).DB()` (`wallet/internal/db/kvdb/driver.go:119`)

Introduced by the same commit, and it has **zero callers anywhere**. Every
`.DB()` use in `wallet/internal/db/itest/` is on a SQL store — those call
`BeginTx` and `ExecContext` on a `*sql.DB`, whereas this method returns
`walletdb.DB`.

Straight deletion, no replacement needed. Introduced-and-never-used is a
stronger version of the introduce-then-remove problem, so this is the clearest
win in the whole task.

*(Method note: an earlier check of this used `-S 'func (s \*Store) DB() ...'`
inside double quotes, which passes a literal backslash to git and silently
matches nothing. It produced a false "pre-exists at base" reading. Confirmed by
direct inspection of `sql-wallet:wallet/internal/db/kvdb/driver.go` that the
method is absent at base and introduced in the stack.)*

### H3. `UgVTm` — stop supporting kvdb wallet *creation* (DECIDED: creation stays)

"I think we should stop supporting creating kvdb wallet — we only need to
support loading them?" Larger than H1/H2, and it now interacts with
[[Task 311]]: if kvdb creation goes away, 311's "new kvdb wallets use single
encryption" case disappears entirely, and 311 reduces to the conversion of
existing wallets. That would simplify both.

Worth deciding before 311 is implemented, since it changes 311's scope. Not
scheduled here.

## Disposition summary

| Thread | PR | Verdict | Action |
| --- | --- | --- | --- |
| `U1T6T` | 1272 | confirmed defect, **merge-blocking** | call `m.lock()`; extend race test |
| `U1VsY` | 1291 | confirmed defect | `beginCreate` guard in `Load` |
| `U1V9E` | 1291 | confirmed defect | hold guard across evict + close |
| `U1H_s` | 1272 | correct | doc + `TODO(yy)` |
| `U1Wts` | 1272 | correct | `TODO` naming the follow-up PR |
| `U1UGB` | 1272 | premise wrong, concern valid | fix churn; correct the framing in the reply |
| `U1Vqn` | 1272 | owner directive | reply pointing at [[Task 310]] |
| `U06ol` | 1272 | reject | reply with evidence; superseded by [[Task 311]] |
| `UgPWI` | 1291 | **reopened** | delete `IsMissingStore`, sentinel instead (H1) |
| `UgRON` | 1291 | **reopened** | delete dead `(*kvdb.Store).DB()` (H2) |
| `UgVTm` | 1291 | open decision | interacts with [[Task 311]] scope (H3) |

## Cost and sequencing

A, C, D, E, F touch #1272; B, H1, H2 touch #1291. Any #1272 change forces a
rebase of #1291 and #1292, so all three PRs re-push and CI re-runs on all three.
Bundle everything in one pass rather than pushing per finding.

Fold each fix into its owning commit — no standalone review-fix commits — and
re-run the full gauntlet at each of the three heads: per-commit `go vet ./...`,
the four pinned golangci passes (`make lint-check`, docker, v2.4.1 — the local
binary is a different version and reports a different finding set),
`fmt-check`/`sqlc-check` in a throwaway worktree, and `-race` on the touched
packages.

If only one thing ships, ship **A**: it breaks a security invariant and the fix
is a single call. H2 is the cheapest.

## References

- [[Task 308]] — the round that produced these heads.
- [[Task 309]] — field-surface cleanup; C is a step toward its candidate 6.
- [[Task 310]] — deprecated-file cleanup; owns F.
- [[Task 311]] — single-encryption invariant; makes C's removal possible,
  supersedes G, and its scope depends on H3.
