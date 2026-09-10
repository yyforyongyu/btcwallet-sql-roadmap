# Task 348 — watch PSBT change before returning funding

Component: **PsbtManager and AddressManager**.

Type: **Bug fix**.

Dependencies: **[Task 344][t344], [Task 356][t356], [Task 402][t402],
[Task 406][t406], and [Task 425][t425]**.

## Rationale

After Task 425 isolates shared authoring from wrapper effects, route PSBT
change using the durable script that Task 356 committed through
Task 344, and return the funded result only after live delivery. Maintained
modern kvdb keeps its legacy count-one change-allocation path until Task 353
retires that backend.

## Details

On registration failure or cancellation, invoke Task 406's private exact-set
cleanup before returning. Immediately before success, verify the full selected
set is active, unexpired, and owned by `FundIntent.LeaseID`. Do not send a
modern-kvdb wallet through SQL-only `NewBulkAddresses`; preserve its existing
count-one allocation and watch flow without adding watch metadata.

Exclusions: direct CreateTransaction delivery, change viability, dust/size
arithmetic, descriptor classification, input selection, token generation, and
lease-release implementation. Shared authoring is owned by Task 425 and cannot
perform this watch or lease cleanup on the wrapper's behalf.

## Suggested fix

Consume only Task 356's committed script for SQL PSBT change, then perform
Task 402-backed registration here. Preserve kvdb/Neutrino watch paths and
perform no second allocation or Store mutation. Gate publication on an exact
funding-set verification; route every pre-return post-funding failure through
Task 406's private cleanup.

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

PSBT success is gated on live registration of durable change and
last-moment exact lease ownership; pre-return cleanup stays private and
successful callers retain existing `ReleaseOutput`. SQL btcd/bitcoind uses
Task 402; modern kvdb and Neutrino retain their existing watch paths.

[t344]: 344-bulk-address-allocation.md
[t356]: 356-defer-change-script-allocation.md
[t402]: 402-replay-live-watch-obligations.md
[t406]: 406-psbt-funding-lease-release.md
[t425]: ../archive/tasks/425-separate-transaction-authoring-wrappers.md
[t439]: 439-psbt-funding-itests.md
