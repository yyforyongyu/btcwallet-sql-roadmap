# Task 394 — exclude key-only locators from wallet ownership

Component: **AddressStore ownership resolution and transaction ingestion**.

Type: **Gap fix**.

Dependencies: **[Task 345][t345]**.

Superseded before implementation on 2026-08-30. The accepted `NoChainSync`
design disables automatic chain synchronization without changing managed
address ownership, eliminating this ownership-exclusion outcome.
[Task 395][t395] owns the separate remaining allocation outcome and is not a
replacement for the eliminated policy.

## Rationale

A persisted key-only locator may map to a normal account child,
but it is not a receiving script, credit, balance, recovery target, or watched
output. Current ownership resolution cannot make that distinction.

## Details

- Exclude key-only account children from `ResolveOwnedAddresses` and every
  equivalent SQL ownership query used by maintained transaction ingestion.
- When one transaction contains both key-only and watch-chain matches, retain
  the transaction and all legitimate credits/debits/status/sync effects while
  treating only the key-only output as non-owned.
- Ensure key-only matches never create UTXOs, balances, usedness, watched
  outpoints, or change provenance.
- Keep recovery query filtering in Task 398 and
  Task 399; keep allocation in Task 395.

## Suggested fix

Join the owning account mode in the batched SQL resolver and
filter at that single Store ownership boundary rather than teaching each
transaction caller a second policy.

## Verification

SQLite and PostgreSQL mixed-output ingestion fixtures prove
the transaction's non-key-only effects survive, the key-only output creates no
wallet ownership fact, and the batched resolver remains wallet-scoped.

## Definition of done

Key-only child scripts are uniformly non-owned for
maintained SQL ingestion without rejecting their containing transaction or
duplicating ownership policy in callers.

[t345]: 345-durable-watch-registration.md
[t395]: 395-key-only-locator-allocation.md
