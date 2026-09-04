# Task 393 — guard receiving APIs for NoChainSync accounts

Component: **AddressManager receiving APIs and AddressStore account lookup**.

Type: **Gap fix**.

Dependencies: **[Task 341][t341], [Task 345][t345], and [Task 392][t392]**.

## Rationale

Receiving APIs promise chain tracking or rely on observed address use. A
`NoChainSync` account intentionally disables that work, so returning a normal
receiving address would make the public contract false and could hide funds.

## Details

- Reject `NewAddress` and `GetUnusedAddress` for a `NoChainSync` account before
  derivation, counter access, Store mutation, watch persistence, or chain I/O.
- Use Task 378's public unsupported-operation error rather than adding a new
  receiving-specific sentinel.
- Task 344 owns the equivalent guard for `NewBulkAddresses`. Task 395 later
  replaces only the `NewAddress` rejection with its non-watching key
  allocation behavior.
- Do not change ownership, signing, transaction ingestion, or recovery.

## Suggested fix

Check the persisted account policy at the existing AddressManager selection
boundary. The exact placement is flexible as long as rejection precedes every
side effect.

## Verification

For both SQL dialects, prove both APIs return the expected public error and
perform no derivation, counter mutation, watch persistence, or chain call for
a `NoChainSync` account. Ordinary accounts remain unchanged.

## Definition of done

Both existing receiving APIs fail closed for `NoChainSync` accounts before
side effects, leaving Task 395 as the sole task that enables count-one key
allocation for those accounts.

[t341]: ../archive/tasks/341-public-role-contract.md
[t345]: 345-durable-watch-registration.md
[t378]: 378-account-manager-error-boundary.md
[t392]: 392-discovery-mode-account-creation.md
