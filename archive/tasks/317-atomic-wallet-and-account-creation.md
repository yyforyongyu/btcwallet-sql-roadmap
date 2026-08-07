# Task 317 — create a wallet and its default accounts in one transaction

**Status:** deferred, not scheduled. Split out of [[archive/315-rebuild-minimal-dual-backend-itest-stack.md]] (archived) §4.

## Why deferred

Less is missing than the name suggests. Verified at base `2ea76dfc5`:

- `sqlite.Store.CreateWallet` already wraps `db.CreateWalletWithOps` in a single
  `s.execWrite` — wallet row, wallet secrets, and initial sync state are already
  written in one SQL transaction;
- base kvdb creation is already one `walletdb.Update` over the waddrmgr and
  wtxmgr namespaces.

What is **not** transactional is **account** creation: default accounts and
initial imported accounts are written after the wallet commits, so a failure
between them leaves a wallet with no accounts. The first Manager itest never
derives an address, so it does not reach that gap.

Measured cost of closing it in the previous attempt: **+264 all-file / +163
production-Go** above the non-aggregate checkpoint.

## What to do

Extend both backends' existing create transaction to cover accounts:

- `createWalletAggregateOps` in both dialects — and place it **after**
  `createWalletOps` is fully declared. The previous attempt wedged it between
  `createWalletOps`'s godoc and its declaration in *both* dialects;
- `DefaultAccountName`, `DefaultAccountScopes` and `InitialAccounts` on
  `db.CreateWalletParams`;
- account key derivation inside the transaction. The previous attempt used a
  create-time derivation callback on the Store contract and a transaction-local
  temporary waddrmgr manager on the kvdb side — both were judged too invasive.
  Find a shape that does not put a callback on `db.Store`.

Do not reintroduce create manifests, partial-state markers, replay, or
authentication of existing state — those are 318's subject and were rejected.

## Definition of done

1. A failure during account creation leaves **no** wallet row and no namespace —
   proven by a fault-injection test on both dialects, falsified.
2. `db.Store` gains no derivation callback.
3. `createWalletAggregateOps` does not separate any declaration from its godoc.
4. Measure the delta against the branch it lands on; it should not exceed ~300
   all-file lines.

## Evidence

`archive/314-post-review-completion-of-the-three-prs.md` §2/§2a/§2c is the
executed version; commits `1eb3dd8d8` (SQL) and `65d2db689` (kvdb) on
`refs/backup/t314-original/*`.

## Carried forward from the delivered stack (codex, 2026-08-02)

**The funded test helper cannot serve SQLite.** It depends on a default account
existing at creation, which SQL does not seed. Keep the helper kvdb-only until
account genesis lands here — do not add default accounts or a funded SQLite
itest as a workaround.
