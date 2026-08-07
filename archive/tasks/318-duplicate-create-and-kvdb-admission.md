# Task 318 — duplicate-create semantics and restart-stable kvdb identity

**Status:** deferred, not scheduled. Split out of [[archive/315-rebuild-minimal-dual-backend-itest-stack.md]] (archived) §4.

## Why deferred

315 creates a wallet by calling the backend's existing create operation once and
returning its duplicate or constraint error. That is base behaviour and it is
enough for the itest, which creates one wallet on a clean database.

Two pre-existing gaps stay open behind it.

## Gap 1 — creating over an existing wallet

The modern `Manager.Create` does not establish, before writing, that the caller
is entitled to the wallet that already exists. Depending on backend, a second
create against an existing name can mutate persisted state under a **new
passphrase** rather than failing cleanly.

The previous attempt fixed this the long way — a persisted create manifest, then
partial-state recovery, then retry authentication, then an insert-only rewrite
that deleted most of it. The net of all that churn was a simple rule, and that
rule is what should be implemented directly:

> An existing artifact is an error. Never a repair, a replay, or a retry.

Decide and document one contract — a distinct `ErrWalletExists` versus the raw
backend constraint error — and apply it identically on both backends.

## Gap 2 — kvdb accepts any wallet name

kvdb is a single-wallet legacy backend, and the owner's ruling is to keep it
that way. But the adapter accepts *aliases* silently: base
`kvdb.Store.GetWallet` does not validate the requested name against the
underlying wallet and echoes back whatever it was given. The behaviour is
documented in a NOTE at base (`wallet/internal/db/kvdb/walletstore.go:67`), so
this is pre-existing, not a regression.

Reject a second name deterministically — but note the constraint that shapes the
fix: **legacy kvdb persists no wallet name at all.** waddrmgr has no such
concept, which is precisely why `GetWallet` can only echo back what it was
given. So there are two different guarantees available, and they cost very
different amounts:

- **in-process** — the Manager's one-entry `wallets` map rejects a second alias
  within a process. **Task 315 §5.1 already implements this**, because its
  Manager-owned single bound address manager forced it; do not implement it a
  second time here;
- **restart-stable** — requires persisting a wallet identity in the kvdb
  namespace. That is new on-disk metadata for a backend being retired
  ([[323-kvdb-to-sql-wallet-migration.md]]), so it needs separate approval and
  must not be smuggled in as an "adapter lookup".

Decide which is being promised before implementing, and document it. Do **not**
revive `Manager.kvdbWallet` for either.

`ListWallets`/`IterWallets` on the kvdb Store should either report the one real
wallet or stay explicitly unimplemented — not silently succeed.

## Definition of done

1. A second `Create` for an existing wallet fails on both backends with one
   documented error, and provably mutates nothing — falsified test.
2. **Only if** new persisted kvdb identity metadata is separately approved: a
   `Load` under a name that is not the wallet's own is rejected **across a
   restart**. The test must close and recreate the Manager before using the
   second name — otherwise it only re-tests Task 315 §5.1's in-process guard,
   which already ships. If that metadata is not approved for a retiring backend,
   drop this item and the task is duplicate-create semantics alone.
3. No manifest, marker, recovery, replay, or retry-authentication machinery.
4. No `Manager.kvdbWallet`.

## Evidence

`archive/300-INDEX.md` ("kvdb supports exactly one wallet", owner 2026-07-30) and
`archive/313-single-encryption-revision-of-the-three-prs.md` R9. The
insert-only commit `ecfdd95d3` on `refs/backup/t314-original/*` shows the
end-state rule after the churn was removed.
