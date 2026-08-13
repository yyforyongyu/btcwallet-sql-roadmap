# Task 393 — reject receiving allocation for key-only accounts

Component: **AddressManager receiving entry points and AddressStore account
lookup**.

Dependencies: **[Task 341][t341] and [Task 345][t345]**.

## Rationale

Once key-only accounts can exist, current receiving methods
would derive an address and present it as wallet-owned. Rejection is a public
AddressManager policy independent of key allocation and transaction ingestion.

## Details

- Load the immutable account mode with every receiving-address selection.
- Reject `NewAddress`, `GetUnusedAddress`, and future receiving-batch requests
  for `key-only` before a counter, derivation callback, Store write, or chain
  call.
- Return one wallet-owned stable error from the public boundary.
- Do not add `NewKey`, watch obligations, batch allocation, or ownership
  filtering.

## Suggested fix

Extend the normalized account lookup used by address
allocation with discovery mode and centralize one pre-mutation receiving-mode
guard in AddressManager.

## Verification

Dependency spies and both SQL dialects prove key-only
receiving requests change no counter or row and perform no derivation or chain
call; `watch-chain` behavior remains unchanged.

## Definition of done

Every receiving allocator fails closed on key-only
accounts before mutation using one wallet-owned error, without implementing a
key allocator or watch delivery.

[t341]: ../archive/tasks/341-public-role-contract.md
[t345]: 345-durable-watch-registration.md
