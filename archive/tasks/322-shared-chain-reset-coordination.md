# Task 322 — coordinate chain resets across wallets sharing one database

**Status:** deferred, not scheduled. Split out of [[archive/315-rebuild-minimal-dual-backend-itest-stack.md]] (archived) §4.

## Why deferred

The Manager itest runs one wallet. The gap only appears with two or more wallets
on one SQL database — which is exactly the configuration the schema was designed
for, so this becomes load-bearing as soon as multi-wallet is exercised.

## The gap

`db.Store.RollbackToBlock(ctx, height)` is **database-wide by contract**, not
wallet-scoped — the signature carries no wallet identity
(`wallet/internal/db/interface.go:489`). Under the one-database-per-Manager
ruling, every wallet in a Manager shares that database. So one wallet reacting
to a reorg rewinds transaction state for **all** of them, including wallets
synced to a different tip or not started at all.

The same shape applies to any other DB-wide sync operation reachable from a
single wallet's chain callbacks.

## What to do

Decide between two shapes before writing code:

1. **Scope the operation** — add wallet identity to `RollbackToBlock` and its
   siblings, making each wallet's rollback affect only its own rows. Larger
   change, but it makes the shared database behave the way the schema implies.
2. **Coordinate the reset** — keep the DB-wide operation and serialize it behind
   a Manager-level owner that quiesces every affected wallet, rolls back once,
   and resumes them. Smaller change to the Store contract, but it introduces the
   cross-wallet coordination the milestone deliberately avoided.

The owner previously stated a shared-chain invariant that reversed an earlier
plan item here; re-confirm it before choosing, since it decides which shape is
correct.

Whichever is chosen, `wallet/runtime_store.go`'s TODO on this subject must be
replaced by the implemented contract, not merely reworded.

## Definition of done

The observable contract differs by shape, so pick the matching row — the two are
not interchangeable:

1. **If scoped:** two wallets on one database, one reorging — the other's
   transaction state and sync tip are provably **untouched**. Falsified test.
   **If coordinated:** every affected wallet is quiesced before the rollback and
   resumes at a tip consistent with it; no wallet observes a partially rolled
   back database, and none is left stopped. Falsified test.
2. The `db.Store` godoc states the scope of every rollback-class operation
   explicitly.
3. No TODO left standing on the question.

## Evidence

`archive/307-global-chain-sync.md`; `archive/306-sql-schema-rewrite.md` item 3
(reversed by the owner's shared-chain invariant). The #1291 thread on
`wallet/runtime_store.go:87` states the gap precisely — see
`.lgtm/pivot/t314-thread-dispositions.md`.
