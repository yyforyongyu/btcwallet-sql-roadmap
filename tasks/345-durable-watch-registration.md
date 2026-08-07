# Task 345 — make discovery policy and watch registration durable

Component: **AddressManager and Controller/Syncer**.

Dependencies: **Complete task: [Task 340][t340] plus accepted ADR 0004 and ADR 0005
amendments for live-watch delivery. [`345.sql-discovery`][t345]: [Task 340][t340] only; it does
not wait for those ADRs**.

## Rationale

The planned allocation contract needs a durable key-only mode. Keeping that mode
only in an allocation request would lose it on restart and conflict with
account-wide recovery, while an address whose initial chain registration fails
may later be returned as usable without repair.

## Details

- Choose immutable `watch-chain` or `key-only` discovery mode when the account
  is created. Allocations inherit it and reject any attempted override.
- Add the SQL storage migration first: backfill every preexisting account as
  `watch-chain`, then make discovery mode present and immutable for all account
  rows. Prove the backfill and immutability after reopen before new creation
  paths depend on the field.
- Key-only locators remain derivable/signable but never become wallet credits,
  recovery targets, balances, or watched outputs.
- During transaction ingestion, classify an output matching a key-only locator
  as non-owned. Retain the transaction's other credits, debits, status, and sync
  update; one key-only output must not reject the whole batch. Key-only accounts
  never create watch obligations.
- Reject ordinary receiving-address allocation for key-only accounts. Their
  next durable locator is allocated through [Task 344][t344]'s count-one key operation,
  and an existing locator remains derivable through [Task 337][t337]. Neither path uses
  an API that promises a receivable wallet address.
- Treat every accountless raw address or script import as `watch-chain`. Commit
  its ordered registration obligation and stable delivery ID atomically with
  the import, then let the Syncer deliver it; do not call `NotifyReceived`
  directly from an import path.
- Amend or supersede ADRs 0004 and 0005 before implementation. Remove ADR
  0004's import-triggered targeted historical scan. Clarify ADR 0005's
  database-only language so the import mutation may persist a live-tip
  obligation that the Syncer delivers asynchronously after the call returns.
  Imports remain synchronous database mutations that return immediately after
  commit, with no chain call and no automatic historical scan. The accepted
  replacement must distinguish that live-tip maintenance from an explicitly
  requested recovery rescan. Preserve ADR 0004's current explicit targeted-
  rescan contract for now; [Task 342][t342] owns the later semantic-account selector
  amendment.
- Commit addresses before external registration.
- Return a typed pending result after commit; it must not expose a usable
  watched address until idempotent registration succeeds.
- Delegate registration, retry, and completion to the Syncer. Allocation must
  not call `NotifyReceived` directly or block the address manager on a chain
  backend response.
- Add one idempotent chain primitive that updates only the live watch set and
  scans no history. Neutrino registration must not reuse birthday-based
  `NotifyReceived`. Account-scoped allocation can request [Task 342][t342]'s targeted
  rescan; an accountless raw import requires an explicit full-wallet recovery
  scan when historical discovery is wanted.
- When Neutrino is already performing a historical scan, isolate or defer live
  registration until a live-tip boundary. Delivery must not expand the active
  scan's start height or target set.
- Treat an SQL import or allocation commit error as ambiguous. Re-read the exact
  locator, generation, obligation, and delivery facts before returning pending,
  definite failure, or a wallet-owned indeterminate result.
- Keep chain calls outside Store transactions.
- Name the SQL discovery-mode column, preexisting-account backfill,
  immutability proof, and backend-neutral mode contract
  [`345.sql-discovery`][t345]. Land regenerated SQL and both-dialect reopen tests as a
  schedulable artifact consumed by [Task 335][t335]. Modern-kvdb acceptance may be
  removed by [Task 353][t353] and must not block that milestone.

## Potential fix

Persist immutable account discovery mode plus a per-wallet watch generation and
ordered registration obligation with a stable delivery ID. Startup registers
one coherent Store snapshot, then the Syncer replays later obligations in order
and completes the pending result. Use the same obligation form for raw imports,
which have no account mode to inherit. Keep the mechanism backend-neutral and
avoid a generic write lock. Deliver through a live-tip-only chain operation and
reconcile an ambiguous Store result from the exact obligation facts.

## Done when

Key-only policy survives restart and cannot be changed by allocation.
Every pre-migration SQL account reopens as `watch-chain` before immutability is
enforced.
Ingestion targeting a key-only locator creates no credit, balance, watched
output, or watch obligation. Watched-address failures repair the same candidate,
existing watch targets are seeded before live sync, and no receiving script is
returned unwatched. A blocked or failed chain backend does not block the address
manager, and restart resumes the same durable delivery.
Raw imports use the same non-blocking durable delivery path, and receiving APIs
reject key-only accounts without allocating a locator. The accepted ADR update
removes the import trigger from ADR 0004 and records that automatic live-tip
delivery is asynchronous watch maintenance, not an import-triggered historical
rescan. Import returns after its database commit without waiting on chain I/O,
historical recovery remains explicit, and [Task 342][t342] remains the sole owner of
the later target-selector change. Ambiguous Store outcomes cannot duplicate a
locator or delivery. A barrier-controlled import during an active Neutrino scan
does not expand that historical scan, and a key-only match cannot roll back
other transaction effects.

[t335]: 335-kvdb-to-sql-migration.md
[t337]: 337-imported-xpub-derivation.md
[t340]: 340-durable-database-identity.md
[t342]: 342-recovery-window-validation.md
[t344]: 344-bulk-address-allocation.md
[t345]: 345-durable-watch-registration.md
[t353]: 353-retire-modern-kvdb-backend.md
