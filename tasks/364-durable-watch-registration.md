# Task 364 — deliver durable live-watch registrations

Component: **AddressManager and Controller/Syncer**.

Dependencies: **[Task 345](345-durable-watch-registration.md), plus accepted
ADR 0004 and ADR 0005 amendments for live-watch delivery**.

## Rationale

Persisted discovery mode does not make a newly allocated or imported script
safe to return when its initial chain registration fails. Delivery needs a
durable, retryable boundary that remains separate from historical recovery.

## Details

- Commit each watch-chain allocation or accountless raw import together with
  its ordered registration obligation and stable delivery ID before chain I/O.
- Return a typed pending result after commit and expose no usable watched
  address until idempotent registration succeeds.
- Delegate registration, retry, and completion to the Syncer. Allocation and
  import paths must not call `NotifyReceived` directly or block their Store
  transaction on the chain backend.
- Add one idempotent chain primitive that updates only the live watch set and
  scans no history. Neutrino must not reuse birthday-based `NotifyReceived`.
- Amend or supersede ADRs 0004 and 0005 first. Remove the import-triggered
  targeted historical scan and document asynchronous live-tip maintenance as
  distinct from an explicitly requested recovery rescan.
- Keep imports synchronous database mutations that return after commit, with
  no chain call and no automatic historical scan. Historical discovery for an
  account-scoped allocation may be requested separately under
  [Task 363](363-recovery-scan-work-bound.md)'s recovery contract; an
  accountless import requires an explicit full-wallet recovery scan.
- When a historical Neutrino scan is active, isolate or defer live
  registration until a live-tip boundary without expanding that scan's start
  height or target set.
- Treat SQL import or allocation commit errors as ambiguous. Re-read the exact
  locator, generation, obligation, and delivery facts before returning
  pending, definite failure, or a wallet-owned indeterminate result.
- Seed one coherent persisted watch snapshot before live sync and replay later
  obligations in order. Keep chain calls outside Store transactions.
- Preserve the maintained modern-kvdb clause only while that backend remains;
  [Task 353](353-retire-modern-kvdb-backend.md) may remove it without blocking
  SQL delivery.

## Suggested fix

Persist a per-wallet watch generation and ordered obligation with a stable
delivery ID, then let the Syncer seed the snapshot and replay obligations
through a live-tip-only chain operation. Reconcile ambiguous Store results
from the exact obligation facts and keep the mechanism backend-neutral.

## Verification

- Allocation and raw-import tests prove commit precedes chain I/O, blocked or
  failed registration returns pending without exposing a usable script, and
  restart repairs the same delivery.
- Ambiguous Store outcomes reconcile without duplicating a locator or delivery,
  and ordered replay cannot drop an accepted obligation.
- Barrier tests prove live registration during an active Neutrino scan does
  not change that scan's start height or targets.
- The accepted ADRs distinguish asynchronous live-tip maintenance from
  historical recovery and keep import return independent of chain I/O.

## Definition of done

- Every watch-chain allocation and raw import has one durable, idempotent,
  restart-safe live-watch delivery before its script becomes usable.
- Live registration never silently starts or widens a historical scan.
- The PR adds no generic write lock, automatic import rescan, or key-only watch
  obligation.
