# Task 408 — watch direct transaction change before return

Component: **TxCreator and AddressManager**.

Type: **Bug fix**.

Dependencies: **[Task 344][t344], [Task 356][t356], [Task 402][t402], and
[Task 425][t425]**.

## Rationale

After Task 425 isolates shared authoring from wrapper effects, route direct
CreateTransaction change using the durable script that Task 356
committed through Task 344, and return an AuthoredTx only after live delivery
succeeds. Maintained modern kvdb keeps its legacy count-one change-allocation
path until Task 353 retires that backend.

## Details

Allocate/watch nothing for absent change. On watch
failure or cancellation, return no result while preserving the committed
SQL address for Task 402's startup reconstruction. Keep modern kvdb on its
existing count-one allocation and watch flow, not SQL-only `NewBulkAddresses`.
Direct TxIntent has no funding lease.

Exclusions: FundPsbt, funding leases, dust/fee calculation, descriptor
classification, and adding ownership tokens to TxIntent. Shared authoring is
watch-neutral and cannot perform this wrapper-owned delivery.

## Suggested fix

Consume only Task 356's committed SQL script and perform Task 402-backed
registration here, without another allocation or Store mutation. Preserve
kvdb/Neutrino watch paths. Make CreateTransaction finish watch delivery
before publishing and returning its AuthoredTx.

## Verification

Direct creation gates success on watch delivery. SQL cancellation leaves the
committed address; maintained modern-kvdb fixtures preserve the legacy
count-one allocator and never call `NewBulkAddresses`. Zero/dust paths create
no address or registration, and tests assert no lease API call.

## Definition of done

Every returned direct AuthoredTx with change has that
change registered. SQL btcd/bitcoind uses Task 402; modern kvdb and Neutrino
retain their existing watch paths. No lease semantics or duplicate allocation.

[t344]: 344-bulk-address-allocation.md
[t356]: 356-defer-change-script-allocation.md
[t402]: 402-replay-live-watch-obligations.md
[t425]: ../archive/tasks/425-separate-transaction-authoring-wrappers.md
