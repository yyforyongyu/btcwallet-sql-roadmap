# Task 344 — add atomic address and key allocation

Component: **AddressManager and key allocation**.

Dependencies: **[Task 337][t337] and
[Task 364](364-durable-watch-registration.md)**.

## Rationale

Repeated count-one allocation can interleave, and an invalid BIP32 child can
wedge a branch because the failed index never advances.

## Details

- Positive count means the number of valid addresses returned.
- Define one small public maximum and reject zero or excess before mutation.
- Keep bulk address allocation receiving-only and reject key-only accounts
  before it changes a counter. A receiving result always follows
  [Task 364](364-durable-watch-registration.md)'s watch-registration contract;
  there is no `SkipChainWatch` request override.
- Make `NewAddress` the exact-account oldest-unused operation. It returns the
  oldest persisted unused receiving address and delegates to a receiving
  count-one batch only when no candidate exists. If that candidate has an
  outstanding [Task 364](364-durable-watch-registration.md) obligation, retry
  or await the same delivery rather than allocating a replacement or exposing
  it early.
- Make `NewBulkAddresses` the force-allocation operation. It always advances
  the selected receiving branch by the requested count and never substitutes
  an older unused candidate.
- Return wallet-owned ready-or-pending results with public `AddressInfo` in
  committed order. A pending result identifies completion but exposes no usable
  receiving address until the complete operation-specific obligation is
  delivered.
- Add one count-one key allocation for key-only accounts. It selects an exact
  account and branch, atomically consumes invalid children and advances the
  durable counter, and returns only a wallet-owned public key and locator. It
  creates no receiving result or watch obligation. [Task 337][t337] remains the pure
  derivation path for an already selected locator.
- Prepare derivation and encryption outside the write transaction.
- Persist consumed invalid children and terminal exhaustion.
- Select the account by stable identity, not only address type.
- Use the BIP32 non-hardened boundary as the terminal address-index limit.
- On a successful batch, persist skipped invalid children with the final
  counter, ordered watch obligation, and generation/delivery record in the same
  atomic mutation. If terminal exhaustion prevents a full batch, commit only a
  terminal exhausted marker, return no addresses, and persist no watch
  obligations.
- Treat a valid derived child whose script is already owned by an accountless
  raw import as a consumed, non-returnable collision. Continue deriving until
  the requested number of fresh scripts is filled or the branch exhausts.
  Reconcile a raw import racing the batch through the same write CAS and
  wallet-wide script uniqueness constraint; never relabel the raw owner or
  return its script as the derived account's address.
- Return one batch-level typed pending result after commit. Withhold the entire
  ordered address list until
  [Task 364](364-durable-watch-registration.md) completes registration;
  delivery failure exposes no usable partial batch.

## Suggested fix

Add one Store batch operation that CASes the expected branch counter and inserts
the complete ordered candidate set, final counter, and durable watch obligation.
`NewAddress` first queries the oldest unused candidate and reuses its existing
delivery record, delegating to a receiving count-one batch only when none
exists. `NewBulkAddresses` always invokes the allocating path. Reuse its counter
and invalid-child machinery for a separate key-only count-one operation, but
omit address delivery and watch facts from that result. Reconcile an ambiguous
commit using the exact candidate, counter, discovery mode, and applicable
delivery facts, not a generic create manifest. Complete one receiving pending
result only after the ordered batch obligation is delivered. Use operation-
specific insert results so a concurrent raw-script collision consumes its
child index and refills the batch inside the same bounded allocation operation.

## Verification

Concurrent batches never overlap, invalid children cannot wedge allocation,
results are ordered, exhaustion returns a typed zero-result error while its
terminal marker prevents retry loops, and an ambiguous result is reconciled
without duplicating either an address or watch delivery. kvdb, SQLite, and
PostgreSQL share the same bounds and exhaustion behavior, and key-only receiving
batches fail before changing counters or creating delivery rows. Concurrent
key-only count-one allocations return distinct committed locators, create no
delivery rows, and resume from the durable counter after restart. Registration
failure returns no usable receiving address while restart repairs the same
committed batch. Raw-import-first and barrier-controlled concurrent-import
fixtures consume the colliding child, preserve the raw owner, and return the
requested count from later fresh children without overlapping another batch.
`NewAddress` returns the oldest unused candidate, retries that candidate's
pending registration without allocating another address, and allocates exactly
one only when none exists. `NewBulkAddresses` always forces the requested new
allocation. Their ready and pending result variants are public and
externally mockable. The key-only count-one operation advances its durable
locator without creating a watch obligation or receiving result.

## Definition of done

- Receiving batches allocate the requested fresh scripts atomically, persist
  skips and ordered watch obligations, and expose no usable partial result.
- `NewAddress`, `NewBulkAddresses`, and key-only count-one allocation retain
  their distinct reuse, force-allocation, and non-receiving contracts.
- Concurrency, exhaustion, collision, registration, and ambiguous outcomes do
  not overlap batches or wedge durable counters.

[t337]: 337-imported-xpub-derivation.md
