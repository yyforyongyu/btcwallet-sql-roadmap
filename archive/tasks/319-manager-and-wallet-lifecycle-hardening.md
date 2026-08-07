# Task 319 — Manager and Wallet concurrency and lifecycle hardening

**Status:** deferred, not scheduled. Split out of [[archive/315-rebuild-minimal-dual-backend-itest-stack.md]] (archived) §4.

## Why deferred

Most gaps below are pre-existing at `sql-wallet`; the post-delivery review also
found two narrow consequences of the new Manager/Vault wiring. None is reached
by the Manager itest, which creates, starts, mines, and stops one wallet from
one goroutine. The owner's contract for that milestone is explicit: **assume
one Manager, one wallet, and a caller that serializes Start and Stop.**

The previous attempt built the machinery and then deleted most of it — a
lifecycle mutex, terminal state, stop claims/results/requests, an eviction
callback, a close fence, and a publication-coordination protocol. Do not
reproduce that. Fix the specific races, or document them as preconditions.

## What Task 315 already supplies — do not redo it

Read this first; two gaps this task used to list no longer exist in the
rebuilt shape:

- **`awaitCreate` is deleted** (315 §4), so the Create-versus-Load race stated
  in terms of it has no referent. If a Create/Load race survives the rebuild,
  restate it against the rebuilt code before implementing anything;
- **cache eviction is deleted** (315 §4) and `Wallet.Stop` closes nothing, so
  "reload after eviction opens a second pool" cannot occur;
- **kvdb in-process second-name rejection ships in 315 §5.1** — the
  `len(m.wallets) != 0` guard under the existing mutex. Not this task, and not
  [[318-duplicate-create-and-kvdb-admission.md]] either;
- 315 takes the existing mutex around **cache lookup, that kvdb check, and
  publication only** — deliberately *not* across backend Create/Open or Wallet
  assembly, which is what leaves the races below open.

## The gaps

**Concurrent `Load` of one name.** Two callers can both miss the cache, open
independent store handles, and publish different `Wallet` instances; the
overwritten instance stays live but untracked. `Manager.Load` has zero
production callers today, which is why this is deferrable.

**Concurrent Start/Stop.** Base leaves these caller-serialized. Whether the
public contract should become internally safe is the real question; answer it
before writing code.

**Use-after-close.** `Manager.Close()` closes the Store with no fence, so a
call arriving afterwards hits a closed handle. 315 makes "the caller has stopped
its wallets" a documented precondition rather than an enforced one.

**Concurrent kvdb loads of different names.** The legacy backend keeps its one
live Store, address manager, and wallet name in mutable fields. `Manager` calls
the backend outside its map lock, so two callers loading different names can
both observe an empty backend, open two address managers, and then overwrite
the retained fields. This races and leaks the first address manager. Per-name
serialization is insufficient: kvdb supports one wallet for the entire
Manager, so either serialize all kvdb admission or document Manager-wide
serialization.

**Repeated Unlock does not refresh the timer.** The legacy Vault returns
`ErrVaultUnlocked` without revalidating a passphrase. `handleUnlockReq` returns
on that sentinel before resetting the auto-lock timer, unlike the pinned-base
waddrmgr path. Decide the public contract explicitly: either make Unlock
idempotent and refresh the timer, or revalidate to preserve the old wrong-
passphrase behavior. Do not expose an internal-only sentinel accidentally.

## What to do

Pick the contract first, then implement the minimum that delivers it:

- if the contract stays "caller serializes", the work is **documentation plus a
  race-detector test that proves the documented usage is safe** — and nothing
  else;
- if the contract becomes "internally safe", serialize build-and-publish for a
  single name (check-and-insert, loser cleanup, or singleflight), and give
  `Close` a fence — but scope it to those two, and measure. Widening 315's
  admission mutex to span backend Create/Open and Wallet assembly is one
  candidate; do **not** restore `awaitCreate`, eviction callbacks, or the
  publication-coordination protocol the rebuild deleted.

Handle the two post-delivery findings independently of that broader choice:

- protect the one-wallet kvdb backend across its full create/load/close state
  transition, or make Manager-wide serialization an enforced precondition;
- add focused repeated-Unlock tests for timer refresh and wrong-passphrase
  behavior before choosing where the idempotency belongs.

Do not restore the terminal-`Stop` model: base `Stop` is restartable and a
stopped wallet stays in the Manager cache. That is settled.

## Definition of done

1. The chosen contract is stated in `Manager`'s and `Wallet`'s godoc.
2. Race tests under `-race` for whichever behaviour is promised.
3. Repeated Unlock has explicit, tested timer and authentication semantics.
4. No terminal state, no `ErrWalletStopped`, no eviction callback.

## Carried forward from the post-delivery GitHub review (2026-08-03)

- #1291 `manager_kvdb.go`: concurrent different-name loads can overwrite the
  backend binding and leak the first address manager.
- #1272 `controller.go`: re-unlocking returns before applying the requested
  timeout and changes the pinned-base authentication behavior.

## Evidence

`archive/300-INDEX.md` decisions 1/1b/1c; `archive/314-post-review-completion-of-the-three-prs.md`
§1/§1a. Unresolved threads on #1291 (`manager.go:1034`, `:1053`,
`controller.go:658`) describe three of these gaps in the reviewers' own words —
see `.lgtm/pivot/t314-thread-dispositions.md`.
