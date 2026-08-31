# Task 348 — watch PSBT change before returning funding

Component: **PsbtManager and AddressManager**.

Type: **Bug fix**.

Dependencies: **[Task 344][t344], [Task 406][t406], and [Task 425][t425]**.

## Rationale

After Task 425 isolates shared authoring from wrapper effects, route PSBT
change for SQL wallets through Task 344's durable watched allocation and return
the funded result only after delivery. Maintained modern kvdb keeps its legacy
count-one change-allocation path until Task 353 retires that backend.

## Details

On registration failure or cancellation, invoke Task 406's private exact-set
cleanup before returning. Immediately before success, verify the full selected
set is active, unexpired, and owned by `FundIntent.LeaseID`. Do not send a
modern-kvdb wallet through SQL-only `NewBulkAddresses`; preserve its existing
count-one allocation and watch flow without adding SQL durable-obligation
semantics.

Exclusions: direct CreateTransaction delivery, change viability, dust/size
arithmetic, descriptor classification, input selection, token generation, and
lease-release implementation. Shared authoring is owned by Task 425 and cannot
perform this watch or lease cleanup on the wrapper's behalf.

## Suggested fix

Consume Task 344's watched allocation for SQL PSBT change and the retained
count-one allocator for modern-kvdb change. Gate publication on delivery and
an exact funding-set verification; route every pre-return post-funding failure
through Task 406's private cleanup.

## Verification

Every returned funded PSBT has watched change and an active exact set owned by
the caller's lease ID. SQL fixtures use Task 344, while maintained modern-kvdb
fixtures preserve the legacy count-one allocator and never call
`NewBulkAddresses`. Registration failure returns no packet and either releases
the attempt's complete set or matches shared `ErrIndeterminateCommit`.
No-change funding performs no watch registration.
[Task 439][t439] owns shared public integration coverage for this final watch,
lease-cleanup, and success contract on every maintained backend.

## Definition of done

PSBT success is gated on durable watch delivery and
last-moment exact lease ownership; pre-return cleanup stays private and
successful callers retain existing `ReleaseOutput`. SQL uses Task 344, while
maintained modern kvdb retains its count-one change-allocation path until Task
353.

[t344]: 344-bulk-address-allocation.md
[t406]: 406-psbt-funding-lease-release.md
[t425]: ../archive/tasks/425-separate-transaction-authoring-wrappers.md
[t439]: 439-psbt-funding-itests.md
