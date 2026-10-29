# Task 466 — verify final oldest-unused allocation

Component: **AddressManager final count-one integration tests**.

Type: **Test**.

Dependencies: **[Task 396][t396]**.

## Rationale

[Issue #1250][issue1250] requires AddressManager integration coverage. Task 431
owns baseline allocation and lookup coverage, not this task's fixtures.
Task 396 replaces the SQL count-one semantics and removes `GetUnusedAddress`.
PR #1407 now proves first allocation and funded-child rotation after reopen;
PR #1402 adds SQL Store ordering/concurrency coverage. The remaining public
proof is repeated unused-address reuse, oldest selection among several
persisted unused children, and kvdb force-next behavior. It does not require
Task 395's separate `AllocateNextKey` API.

## Details

- Extend Task 431's merged address suite. Reuse its first-allocation and
  used-to-next/reopen cases rather than duplicating them; add the missing
  repeated-unused and oldest-of-several assertions. Preserve backend-common
  assertions and do not repeat production-owned mechanical caller migration.
- Reuse the existing SQL Store ordering, exhaustion, and concurrency evidence
  without reproducing its matrix at the public boundary. Keep this task's
  missing public allocation proof distinct from those storage-level tests.
- On SQLite and PostgreSQL, require `NewAddress` for a chain-synchronized
  account to return the oldest persisted unused address without advancing the
  branch. After funding that address, require the next call to rotate to the
  next oldest unused result or allocate one when none remains.
- Keep failed live registration and startup watch reconstruction in Tasks
  396 and 402's colocated tests. This public integration task observes only a
  completed `NewAddress` call and adds no delivery bookkeeping or fault seam.
- In a separate modern-kvdb case, require each `NewAddress` call to preserve
  the maintained force-next behavior. Do not apply the SQL oldest-unused
  assertion to a backend that deliberately cannot represent it.
- Restrict shared three-backend assertions to common public address metadata,
  point/list agreement, errors, and durable cleanup. Keep the divergent
  allocation expectation in the two explicit backend branches.
- Keep `AllocateNextKey` results in Task 465, batch allocation in Task 461,
  and production fault injection in Tasks 395 and 396. Do not restore
  `GetUnusedAddress` or add a compatibility alias.
- Extend existing `itest/address_manager_test.go` and its funding, point/list,
  and reload fixtures. Add no separate baseline address matrix or helper file.

## Suggested fix

Extend the existing address cases with two-dialect repeated-unused and
oldest-of-several assertions and one modern-kvdb force-next case. Reuse the
existing funded-child rotation and reopen proof without changing common
assertions or repeating production caller migrations.

## Verification

- Run identical public repeated-unused and oldest-of-several assertions on
  SQLite and PostgreSQL. Reuse the existing first-allocation, funded rotation,
  and reopen cases instead of creating another matrix.
- Run the modern-kvdb force-next case separately and prove each successful
  call returns a fresh result with durable branch progress.
- Verify `GetUnusedAddress` is absent and the change adds no compatibility API,
  direct Store assertion, or unrelated allocation coverage.

## Definition of done

- Final SQL oldest-unused and modern-kvdb force-next behavior have focused
  public coverage in this separate task.
- Shared assertions cover only behavior common to all maintained backends, and
  the documented divergence is explicit.
- No new allocation method, compatibility alias, arbitrary file, or duplicated
  Task 431 case is introduced.

[issue1250]: https://github.com/btcsuite/btcwallet/issues/1250
[t395]: ../archive/tasks/395-key-only-locator-allocation.md
[t396]: ../archive/tasks/396-oldest-unused-address.md
[t431]: ../archive/tasks/431-address-allocation-lookup-itests.md
