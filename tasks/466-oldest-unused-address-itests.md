# Task 466 — verify final oldest-unused allocation

Component: **AddressManager final count-one integration tests**.

Type: **Test**.

Dependencies: **[Task 396][t396]**.

## Rationale

[Issue #1250][issue1250] requires AddressManager integration coverage. Task 431
owns baseline allocation and lookup coverage, not this task's fixtures.
Task 396 replaces the SQL count-one semantics and removes `GetUnusedAddress`.
The final SQL and modern-kvdb divergence needs a separate public integration
outcome; it does not require Task 395's separate `AllocateNextKey` API.

## Details

- Test Task 396's final API without waiting for Task 431. Reuse any existing
  address suite, preserve backend-common assertions, and do not duplicate
  production-owned mechanical caller migration.
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
- Extend a suitable existing address integration-test file. If none exists,
  add only the small conventional setup needed here, without implementing
  Task 431's separate matrix. Reuse overlapping setup whichever task lands
  second; do not wait for a particular file or helper to be introduced.

## Suggested fix

Add one two-dialect SQL oldest-unused table and one modern-kvdb force-next case
to the available address suite. Reuse existing funding, point/list, and
reopen helpers without changing backend-common assertions or repeating the
production tasks' mechanical migrations. Task 431 need not land first.

## Verification

- Run identical oldest-unused, used-to-next, empty-branch, and reopen cases on
  SQLite and PostgreSQL.
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
[t396]: 396-oldest-unused-address.md
[t431]: 431-address-allocation-lookup-itests.md
