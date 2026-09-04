# Task 344 — allocate receiving-address batches atomically

Component: **AddressManager receiving batches and SQL AddressStore**.

Type: **Gap fix**.

Dependencies: **[Task 341][t341], [Task 364][t364], [Task 392][t392],
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
- Commit the consumed derivation progress, returned address facts, and one
  durable Task 364 watch obligation atomically on SQLite and PostgreSQL.
- Return a batch only after Task 402 makes its watch live. A failure after
  commit returns no usable batch and leaves enough durable state for retry or
  startup replay.
- Preserve consumed invalid children and raw-import ownership. Return a stable
  wallet-owned `ErrAddressDerivationExhausted` for terminal exhaustion, and
  preserve Task 467's indeterminate-commit identity without reusing a
  potentially committed child.
- Reject `NoChainSync` accounts and modern kvdb with Task 378's
  `ErrAccountOperationUnsupported` before mutation. Task 395 owns count-one
  key allocation for `NoChainSync` accounts.
- Return wallet-owned `AddressInfo` values with truthful imported and
  derivation metadata. Task 461 owns public three-backend integration tests.
- Do not add a request wrapper, delivery handle, recovery API, public
  reservation object, generic Store callback, or broader address type support.

## Suggested fix

Reuse the existing account selection, derivation, AddressStore, and durable
watch mechanisms in one Wallet-owned batch operation. A branch-local
serialization boundary and expected-position Store mutation are likely, but
their representation is not part of the contract.

## Verification

- Exercise counts 1, 100, 0, and 101 and verify validation precedes mutation.
- Race batches on SQLite and PostgreSQL and prove complete, ordered,
  non-overlapping results across reopen.
- Cover invalid children, raw-import collisions, definite rollback, and
  ambiguous durable outcomes without index reuse. Terminal exhaustion matches
  `ErrAddressDerivationExhausted` on both SQL dialects.
- Fail or cancel live watch delivery after commit and prove no usable partial
  batch is returned and startup replay can complete the obligation.
- Verify `NoChainSync` and modern-kvdb requests fail before mutation, while
  Task 461 covers the final public result contract.

## Definition of done

One public force-allocation call returns an ordered all-or-nothing batch only
after its exact durable addresses are live-watched. Concurrency, restart, and
failure cannot overlap batches, lose derivation progress, or expose private
delivery bookkeeping.

[t341]: ../archive/tasks/341-public-role-contract.md
[t364]: 364-durable-watch-registration.md
[t378]: 378-account-manager-error-boundary.md
[t392]: 392-discovery-mode-account-creation.md
[t393]: 393-key-only-receiving-rejection.md
[t395]: 395-key-only-locator-allocation.md
[t402]: 402-replay-live-watch-obligations.md
[t461]: 461-address-batch-itests.md
[t467]: ../archive/tasks/467-indeterminate-commit-identity.md
