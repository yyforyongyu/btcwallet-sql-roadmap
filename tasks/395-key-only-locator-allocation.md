# Task 395 — allocate one durable NoChainSync child

Component: **AddressManager child allocation and SQL branch-counter mutation**.

Type: **Gap fix**.

Dependencies: **[Task 341][t341], [Task 344][t344], [Task 390][t390],
[Task 393][t393], [Task 396][t396], [Task 399][t399], [Task 452][t452], and
[Task 467][t467]**.

## Rationale

Lnd `DeriveNextKey` needs a durable, unique account child. Reading a counter
and deriving without a mutation can return the same locator concurrently,
while the normal receiving path creates chain-watch obligations that a
`NoChainSync` account explicitly disables.

## Details

- Extend Task 396's final `NewAddress` API. Select behavior from the account's
  persisted `NoChainSync` value; do not add a per-call override or another
  public key-family API.
- For a `NoChainSync` account, atomically allocate exactly one external or
  internal child and return its public key with a complete derivation path.
- Persist counter progress and every accepted child so concurrent and
  restarted calls cannot return the same locator. Consume invalid or colliding
  children and persist terminal exhaustion.
- Never retry an ambiguous commit in a way that can reuse a child. Return the
  shared indeterminate-commit error unless exact persisted facts prove the
  result.
- Create no live-watch obligation, chain notification, recovery target, or
  receiving-address promise for this branch.
- Preserve ordinary address lookup, transaction ownership, and signing for
  the allocated child. Preserve modern kvdb behavior; it cannot create the
  SQL-only account policy that selects this branch.
- Reuse existing account counters, managed-address rows, and wallet-owned
  public errors. Do not add a locator table, reservation service, or another
  ownership model.
- Task 465 owns public integration assertions across real backends.

## Suggested fix

Reuse the existing AddressManager allocation and counter mutation for a
non-watching count-one branch. Any local concurrency mechanism is acceptable
if it proves durable uniqueness and unambiguous result classification.

## Verification

- Concurrent SQLite and PostgreSQL calls return distinct durable paths and
  continue doing so after reopen.
- Invalid children, script collisions, exhaustion, and ambiguous outcomes
  never reuse an index or relabel an existing owner.
- The result contains complete public metadata and remains resolvable through
  ordinary lookup, ownership, and signing paths.
- Prove the branch performs no watch persistence, recovery derivation, or
  chain call, while ordinary accounts and modern kvdb retain their behavior.

## Definition of done

`NewAddress` allocates one durable unique child for a `NoChainSync` account
with truthful key metadata and no chain-synchronization side effects or new
public API.

[t341]: ../archive/tasks/341-public-role-contract.md
[t344]: 344-bulk-address-allocation.md
[t390]: 390-custom-scope-account-creation.md
[t393]: 393-key-only-receiving-rejection.md
[t396]: 396-oldest-unused-address.md
[t399]: 399-zero-lookahead-persisted-targets.md
[t452]: 452-key-only-recovery-derivation.md
[t465]: 465-new-address-result-itests.md
[t467]: ../archive/tasks/467-indeterminate-commit-identity.md
