# Task 344 — allocate receiving-address batches atomically

Component: **AddressManager receiving batches and SQL AddressStore**.

Type: **Gap fix**.

Dependencies: **[Task 341][t341], [Task 392][t392],
[Task 393][t393], [Task 402][t402], and [Task 467][t467]**.

## Rationale

Calling a count-one address API repeatedly does not reserve a batch as one
operation. Concurrent callers can interleave, and failures can leave gaps or
durable addresses that were returned before their chain watch became live.
Lnd needs an explicit force-allocation operation for batches while
`NewAddress` becomes the safe oldest-unused receiving operation in Task 396.

## Details

- Add final `NewBulkAddresses(context.Context, AccountSelector, bool, uint32)
  ([]AddressInfo, error)`. The boolean selects the internal branch and the
  count must be between one and exported `MaxBulkAddressCount = 100`.
- Validate count and selector before mutation. Invalid counts match
  `ErrInvalidParam`.
- Allocate exactly the requested number of fresh addresses in committed order
  or return no addresses. Concurrent batches for one account and branch must
  not overlap.
- Atomically commit SQL derivation progress and addresses, not watch metadata.
- Before return, complete Task 402 registration on btcd/bitcoind or existing
  registration/history on Neutrino; no live-only guarantee, adapter fix, or
  new task for Neutrino. Failure preserves committed facts, returning no batch.
- Preserve consumed invalid children and raw-import ownership. Return a stable
  wallet-owned `ErrAddressDerivationExhausted` for terminal exhaustion, and
  preserve Task 467's indeterminate-commit identity without reusing a
  potentially committed child.
- Reject `NoChainSync` accounts and modern kvdb with Task 378's
  `ErrAccountOperationUnsupported` before mutation. Task 395 owns the separate
  `AllocateNextKey` operation for `NoChainSync` accounts.
- Return wallet-owned `AddressInfo` values with truthful imported and
  derivation metadata. Task 461 owns public three-backend integration tests.
- Do not add a request wrapper, delivery handle, recovery API, public
  reservation object, generic Store callback, or broader address type support.

## Suggested fix

Reuse the existing allocation boundary and backend-specific registration.

## Verification

- Exercise counts 1, 100, 0, and 101 and verify validation precedes mutation.
- Race batches on SQLite and PostgreSQL and prove complete, ordered,
  non-overlapping results across reopen.
- Cover invalid children, raw-import collisions, definite rollback, and
  ambiguous durable outcomes without index reuse. Terminal exhaustion matches
  `ErrAddressDerivationExhausted` on both SQL dialects.
- Fail or cancel live watch delivery after commit and prove no usable partial
  batch is returned and a fresh startup watches the committed addresses.
- Verify `NoChainSync` and modern-kvdb requests fail before mutation, while
  Task 461 covers the final public result contract.

## Definition of done

One public force-allocation call returns an ordered all-or-nothing batch only
after backend-appropriate registration. Concurrency, restart, and
failure cannot overlap batches, lose derivation progress, or expose private
delivery bookkeeping.

[t341]: ../archive/tasks/341-public-role-contract.md
[t378]: ../archive/tasks/378-account-manager-error-boundary.md
[t392]: ../archive/tasks/392-discovery-mode-account-creation.md
[t393]: ../archive/tasks/393-key-only-receiving-rejection.md
[t395]: 395-key-only-locator-allocation.md
[t402]: ../archive/tasks/402-replay-live-watch-obligations.md
[t461]: 461-address-batch-itests.md
[t467]: ../archive/tasks/467-indeterminate-commit-identity.md
