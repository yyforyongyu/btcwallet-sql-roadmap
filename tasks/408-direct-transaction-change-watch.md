# Task 408 — watch direct transaction change before return

Component: **TxCreator and AddressManager**.

Dependencies: **[Task 344][t344], [Task 356][t356], and [Task 425][t425]**.

## Rationale

After Task 425 isolates shared authoring from wrapper effects, route direct
CreateTransaction change for SQL wallets through Task 344's durable watched-
allocation primitive and return an AuthoredTx only after registration
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

Have Task 344's watched allocator return the derived SQL script and durable
obligation together, while the modern-kvdb branch retains its existing
count-one allocator. Make CreateTransaction finish the backend-appropriate
watch delivery before returning its AuthoredTx.

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
[t425]: 425-separate-transaction-authoring-wrappers.md
