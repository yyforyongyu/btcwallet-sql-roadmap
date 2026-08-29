# Task 393 — guard receiving APIs for NoChainSync accounts

Component: **AddressManager receiving entry points and AddressStore account
lookup**.

Type: **Gap fix**.

Dependencies: **[Task 341][t341], [Task 345][t345], and [Task 392][t392]**.

## Rationale

Receiving APIs promise chain tracking or depend on observed usedness. Letting
them operate on a `NoChainSync` account would return a receiving address while
silently omitting the synchronization needed to honor that contract.

## Details

- Load `NoChainSync` with every receiving-address account selection.
- Reject the existing `NewAddress` and `GetUnusedAddress` methods for a
  `NoChainSync` account before counter access, derivation, Store mutation,
  live-watch persistence, or a chain call. Reuse Task 378's canonical
  `ErrAccountOperationUnsupported` instead of adding a receiving-specific
  sentinel.
- Leave the future `NewBulkAddresses` guard to Task 344, which introduces that
  method. Task 395 replaces only the `NewAddress` rejection with behavior
  selected from the account's persisted `NoChainSync` property and owns its
  explicit count-one non-watching branch.
- Do not change ownership, signing, transaction ingestion, or recovery here.
- Keep implementation local to the existing AddressManager guard, account
  lookup, source files, and colocated tests. Add no parallel interface,
  arbitrary helper, production file, or standalone test file; justify any
  unavoidable new file against the repository's current layout in the
  implementation PR.
- Document every added code block inline with why it exists and how it works;
  comments that merely restate the code do not satisfy this requirement.
- Give every unit test detailed Arrange, Act, and Assert comments explaining
  its setup, operation, and evidence. Interface mocks embed `mock.Mock`, set
  `On(...).Return(...)` expectations in Arrange with the narrowest cardinality
  (`Once()` for one required call), and call `AssertExpectations(t)` directly
  in Assert without duplicate call assertions or a `require.True` wrapper.

## Suggested fix

Extend the normalized account lookup used by the two existing receiving
methods and centralize one pre-mutation `NoChainSync` guard in AddressManager.
Task 395 later replaces the `NewAddress` branch without adding another method.

## Verification

Dependency spies and both SQL dialects prove `NoChainSync` `NewAddress` and
`GetUnusedAddress` calls change no counter or row and perform no derivation,
watch persistence, or chain call. Ordinary account behavior remains
unchanged, and both guarded entry points return Task 378's wallet-owned error
identity through `errors.Is`. Task 344 tests its own later batch guard.

## Definition of done

Both existing APIs that currently create or return a receiving address fail
closed on `NoChainSync` accounts before mutation. `NewAddress` remains the
single count-one allocation API for Task 395 to extend, while the later batch
guard and ownership, signing, ingestion, and recovery remain in their owning
tasks.

[t341]: ../archive/tasks/341-public-role-contract.md
[t345]: 345-durable-watch-registration.md
[t392]: 392-discovery-mode-account-creation.md
