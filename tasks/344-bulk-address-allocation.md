# Task 344 — allocate receiving-address batches atomically

Component: **AddressManager receiving batches and SQL AddressStore mutation**.

Type: **Gap fix**.

Dependencies: **[Task 341][t341], [Task 364][t364], [Task 393][t393], and
[Task 402][t402]**.

## Rationale

Repeated count-one allocation can interleave, and a failed
invalid child rolls back to the same branch index. lnd also needs a true
force-allocation API selected by exact account identity.

## Details

- Add final `NewBulkAddresses` with a wallet-owned semantic account selector,
  explicit branch, and positive count. Define one small public maximum and
  reject zero/excess before mutation.
- Accept only `watch-chain` accounts. There is no `SkipChainWatch` override.
- Return the requested number of fresh public `AddressInfo` values in committed
  order. Always force allocation; never substitute an older unused address.
- Derive candidates outside the write, then atomically CAS the expected branch
  counter, insert the ordered fresh scripts, persist consumed invalid children
  and terminal exhaustion, and append one operation-specific watch obligation.
- Treat a valid child colliding with an accountless raw import as consumed and
  refill from later children. Serialize a racing raw import through the same
  wallet-wide script uniqueness and CAS without relabeling the raw owner.
- If the branch cannot fill the complete batch, persist only terminal
  exhaustion and return no addresses or obligation.
- Reconcile an ambiguous mutation from exact candidate/counter/obligation
  facts. Never retry from the transport error.
- Return ready only after Task 402 delivers the batch. A typed pending
  result identifies completion but exposes no usable partial list.
- Return unsupported before mutation on modern kvdb. Its existing count-one
  backend behavior remains outside this SQL batch task until retirement, but
  no old Go API shape is retained solely for callers.
- Keep oldest-unused selection and key-only allocation in their own tasks.

## Suggested fix

Add one method-specific SQL batch CAS and public
`NewBulkAddresses` wrapper that binds the complete ordered row set and one
retained Task 364 obligation in the same transaction.

## Verification

Both dialects cover concurrent non-overlap, order, count
bounds, invalid children, raw-import races, exhaustion, restart, ambiguous
outcomes, registration failure, and no partial exposure. Key-only and kvdb
requests fail before mutation.

## Definition of done

One receiving batch atomically commits its exact fresh
scripts, final cursor, skips, and durable obligation and exposes the complete
ordered result only after live delivery.

[t341]: ../archive/tasks/341-public-role-contract.md
[t364]: 364-durable-watch-registration.md
[t393]: 393-key-only-receiving-rejection.md
[t402]: 402-replay-live-watch-obligations.md
