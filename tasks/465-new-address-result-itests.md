# Task 465 — verify durable key allocation results

Component: **AddressManager AllocateNextKey integration tests**.

Type: **Test**.

Dependencies: **[Task 395][t395] and [Task 431][t431]**.

## Rationale

[Issue #1250][issue1250] requires AddressManager integration coverage. Task 431
owns the existing count-one allocation and lookup contract and must remain
unchanged. Task 395 adds `AllocateNextKey` for durable `NoChainSync` key
allocation, which needs focused public integration evidence.

## Details

- Extend Task 431's imported-`wallet` address suite while preserving its
  existing receiving assertions and definition of done.
- On SQLite and PostgreSQL, create a root-derived numbered account with
  `NoChainSync=true` and allocate through `AddressManager.AllocateNextKey`,
  including a purpose-1017 scope and numbered key-family account. Require the
  returned public key and complete derivation metadata to match the selected
  account and branch.
- Use the fixture's account XPub and returned child index to independently
  derive the expected key and address. Check `GetAddressInfo` and list lookup
  retain the same key, path, account membership, and balance after reopen.
- Extend existing concurrency and reopen cases to prove fresh, distinct paths
  even while all earlier allocated children have no observed chain use.
- Modern kvdb cannot represent the persisted `NoChainSync` property. Keep it
  outside the SQL-only allocation case and do not invent a backend-only
  request or synthetic account state.
- Task 391 depends on Task 395 and owns the `AllocateNextKey` adaptation and
  focused tests for path-bound supplied-XPub accounts with `NoChainSync=true`.
  Keep numberless imported-XPub receiving results in Task 344's focused tests.
  Do not import either account-shape outcome into this integration task.
- Match only wallet-owned errors reachable through ordinary public setup.
  Keep terminal exhaustion and ambiguous-commit injection in Tasks 344 and
  395's colocated tests rather than adding a production fault seam.
- Do not repeat Task 431's ordinary address types, oldest-unused behavior,
  missing lookup, or general cursor-durability matrix. Add only the new
  account-shape and result-metadata assertions listed here.
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

Extend existing test functions or table cases in Task 431's address suite
with the SQL key-allocation contract. Reuse its lookup, concurrency, and
reopen helpers; preserve the original receiving assertions rather than
copying them into another test.

## Verification

- Run numbered `NoChainSync` allocation, lookup, concurrency, and reopen cases
  unchanged on SQLite and PostgreSQL.
- Prove the new fixtures add no fabricated derivation path, public API, or
  direct Store assertion.

## Definition of done

- Numbered `NoChainSync` key allocations have focused public coverage without
  changing Task 431's existing contract.
- The modern-kvdb structural exclusion and SQL result metadata are truthful.
- Tests use Task 395's API without introducing another method, result type,
  component, arbitrary file, or duplicated Task 431 case.

[issue1250]: https://github.com/btcsuite/btcwallet/issues/1250
[t395]: 395-key-only-locator-allocation.md
[t431]: 431-address-allocation-lookup-itests.md
