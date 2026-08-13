# Task 395 — allocate one durable key-only locator

Component: **AddressManager key allocation and SQL branch-counter mutation**.

Dependencies: **[Task 341][t341], [Task 393][t393], [Task 394][t394], and
[Task 399][t399]**.

## Rationale

Lnd `DeriveNextKey` needs a durable, concurrent count-one
mutation. Pure derivation after reading a counter can return the same locator
twice, while a receiving allocator would create a false watch obligation.

## Details

- Add one wallet-owned `NewKey` request selecting an exact semantic account and
  branch, and return only public key plus committed locator.
- Require immutable `key-only` mode and branch 0 or 1 before counter access.
- Atomically consume invalid children, persist the selected locator and final
  counter, and persist terminal exhaustion at the non-hardened boundary.
- Resolve an ambiguous commit from the exact locator/counter facts. A retry may
  leave a harmless gap but must never reuse a committed locator.
- Create no receiving address result, watch obligation, chain call, credit, or
  wallet-owned output. Reuse Task 394 if the smallest Store
  representation shares derived-address rows.
- Rely on Task 399's completed mode-aware persisted-target loaders so a locator
  stored in a shared derived-address row cannot enter full or targeted recovery.
- Return unsupported before mutation on modern kvdb.

## Suggested fix

Add one method-specific SQL count-one mutation using the
existing account branch counters, with derivation outside the write and an
expected-counter CAS. Persist only the mode-aware locator facts needed for
lookup and signing.

## Verification

Concurrent calls on SQLite and PostgreSQL return distinct
locators; invalid-child, boundary, restart, and ambiguous-result fixtures never
reuse an index. No test observes an address, ownership row, obligation, or
chain call.

## Definition of done

One key-only call returns one durable unique public
locator with bounded counter progression and no receiving or watch semantics.
Task 399 already excludes its persisted representation from recovery loading.

[t341]: ../archive/tasks/341-public-role-contract.md
[t393]: 393-key-only-receiving-rejection.md
[t394]: 394-key-only-transaction-ownership.md
[t399]: 399-zero-lookahead-persisted-targets.md
