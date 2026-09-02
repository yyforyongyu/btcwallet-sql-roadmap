# Task 408 — watch direct transaction change before return

Component: **TxCreator and AddressManager**.

Type: **Bug fix**.

Dependencies: **[Task 344][t344], [Task 356][t356], and [Task 425][t425]**.

## Rationale

After Task 425 isolates shared authoring from wrapper effects, route direct
CreateTransaction change using the script and durable obligation that Task 356
committed through Task 344, and return an AuthoredTx only after live delivery
succeeds. Maintained modern kvdb keeps its legacy count-one change-allocation
path until Task 353 retires that backend.

## Details

Allocate/watch nothing for absent change. On watch
failure or cancellation, return no result while retaining the durable
registration obligation for retry on SQL. Do not send a modern-kvdb wallet
through SQL-only `NewBulkAddresses`; preserve its existing count-one allocation
and watch flow. Direct TxIntent has no funding lease.

Exclusions: FundPsbt, funding leases, dust/fee calculation, descriptor
classification, and adding ownership tokens to TxIntent. Shared authoring is
watch-neutral and cannot perform this wrapper-owned delivery.

## Suggested fix

Consume Task 356's committed SQL script and durable obligation or its committed
modern-kvdb count-one result without another allocation or Store mutation.
Make CreateTransaction finish the backend-appropriate live watch delivery
before publishing and returning its AuthoredTx.

## Verification

Direct creation gates success on watch delivery. SQL cancellation leaves the
durable retry obligation; maintained modern-kvdb fixtures preserve the legacy
count-one allocator and never call `NewBulkAddresses`. Zero/dust paths create
no obligation, and tests assert no lease API call.

## Definition of done

Every returned direct AuthoredTx with change has that
change registered and delivered. SQL uses Task 344's durable obligation, while
maintained modern kvdb retains its count-one path until Task 353; the PR adds no
lease semantics or duplicate allocation calculation.

[t344]: 344-bulk-address-allocation.md
[t356]: 356-defer-change-script-allocation.md
[t425]: ../archive/tasks/425-separate-transaction-authoring-wrappers.md
