# Task 466 — verify final oldest-unused allocation

Component: **AddressManager final count-one integration tests**.

Type: **Test**.

Dependencies: **[Task 396][t396] and [Task 431][t431]**.

## Rationale

[Issue #1250][issue1250] requires AddressManager integration coverage. Task 431
owns the existing allocation and lookup contract and must remain unchanged.
Task 396 replaces the SQL count-one semantics and removes `GetUnusedAddress`.
The final SQL and modern-kvdb divergence needs a separate public integration
outcome; it does not require Task 395's separate `AllocateNextKey` API.

## Details

- Extend Task 431's final-API suite after Tasks 396 and 431 land. Preserve
  its backend-common assertions and do not duplicate either production task's
  mechanical caller migration.
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
- Extend the repository-conventional address integration-test file introduced
  by Task 431. Add no arbitrary helper, production file, or standalone test
  file; justify any unavoidable new file against the repository's current
  layout in the implementation PR.
- Document only non-obvious test contracts, rationales, invariants, and
  consequences. Keep Arrange, Act, and Assert structure clear without
  mandatory section comments that restate the test code.
- Interface mocks, when needed, embed `mock.Mock`. Set `On(...).Return(...)`
  expectations during Arrange with the narrowest correct cardinality (`Once()`
  for one required call). During Assert, call `AssertExpectations(t)` directly;
  do not duplicate call assertions or wrap the result in `require.True`.

## Suggested fix

Add one two-dialect SQL oldest-unused table and one modern-kvdb force-next case
to Task 431's final-API suite. Reuse its existing funding, point/list, and
reopen helpers without changing backend-common assertions or repeating the
production tasks' mechanical migrations.

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
[t395]: 395-key-only-locator-allocation.md
[t396]: 396-oldest-unused-address.md
[t431]: 431-address-allocation-lookup-itests.md
