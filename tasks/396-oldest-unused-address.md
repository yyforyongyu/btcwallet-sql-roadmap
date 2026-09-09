# Task 396 — make NewAddress return the oldest unused address

Component: **AddressManager count-one semantics**.

Type: **Bug fix**.

Dependencies: **[Task 344][t344]**.

## Rationale

SQL `NewAddress` currently force-allocates while `GetUnusedAddress` finds the
oldest unused candidate. That naming makes it easy for callers to expand the
address gap accidentally. The stable API should use `NewAddress` for the safe
oldest-unused behavior and `NewBulkAddresses` for deliberate force allocation.

## Details

- Give `NewAddress` its final context-aware selector signature and return
  `AddressInfo`. Derive the address type from the selected account.
- On SQL, return the oldest unused address for the selected account and branch.
  If none exists, allocate one through Task 344's batch behavior.
- Do not return an address until any required live-watch registration is
  usable. A pending obligation must not cause replacement allocation.
- Before changing the SQL semantics, migrate only the SQL branches of existing
  wallet-owned callers that require a fresh change address to
  `NewBulkAddresses(count=1)`.
- Remove `GetUnusedAddress` and update maintained callers directly; do not keep
  a compatibility alias.
- Preserve modern kvdb's current force-next `NewAddress` behavior until that
  backend is retired.
- This task owns only the mechanical fresh-address caller migration. It does
  not absorb the PSBT or transaction correctness fixes in Tasks 348, 356, or
  408. Task 395 owns the separate `AllocateNextKey` API, and Task 466 owns
  public integration assertions.
- Preserve the `NoChainSync` receiving rejection in `NewAddress`, including
  its public documentation. Key allocation must not enter the oldest-unused
  lookup or receiving batch fallback.
- Do not add a first-unused pointer, recovery method, result wrapper, or
  parallel public interface.

## Suggested fix

Reuse Task 344's allocation boundary for the SQL oldest-unused lookup and
count-one fallback. The exact synchronization and query structure are not
part of the public contract.

## Verification

- SQL tests cover reuse, rotation after use, an empty branch, pending watch
  delivery, cancellation, and concurrent empty-branch callers without an
  unnecessary second allocation.
- Prove SQL fresh-change callers use the batch API and still request exactly
  one new address.
- Prove modern kvdb continues to force-allocate and never uses the SQL batch
  path.
- Prove all maintained callers use the final API and no compatibility alias
  remains.

## Definition of done

SQL `NewAddress` safely returns the oldest unused address, deliberate fresh
allocation uses `NewBulkAddresses`, and modern kvdb retains its legacy
count-one behavior. `NoChainSync` receiving remains unsupported, without
expanding this task into key allocation or adjacent transaction bugs.

[t344]: 344-bulk-address-allocation.md
[t348]: 348-watched-change-delivery.md
[t356]: 356-defer-change-script-allocation.md
[t395]: 395-key-only-locator-allocation.md
[t408]: 408-direct-transaction-change-watch.md
[t466]: 466-oldest-unused-address-itests.md
