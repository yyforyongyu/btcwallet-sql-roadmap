# Task 465 — verify NoChainSync address results

Component: **AddressManager NoChainSync result integration tests**.

Type: **Test**.

Dependencies: **[Task 395][t395] and [Task 431][t431]**.

## Rationale

[Issue #1250][issue1250] requires AddressManager integration coverage. Task 431
owns the existing count-one allocation and lookup contract and must remain
unchanged. Task 395 adds the independently changeable `NoChainSync` result
that needs one separate public integration-test outcome.

## Details

- Extend Task 431's imported-`wallet` address suite without changing that
  task's original cases or definition of done.
- On SQLite and PostgreSQL, create a numbered `NoChainSync=true` account and
  allocate through the final existing `NewAddress` method. Require the result
  and `GetAddressInfo` to contain its public key and complete derivation path,
  remain visible by address and balance through list lookup after reopen,
  progress uniquely under concurrency, and retain the selected account
  identity.
- Modern kvdb cannot represent the persisted `NoChainSync` property. Keep it
  outside the SQL-only allocation case and do not invent a backend-only
  request or synthetic account state.
- Keep path-bound XPub result coverage in Task 391's focused tests and
  numberless imported-XPub result coverage in Task 344's focused tests. Do not
  import either production outcome into this public integration task.
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

Add one SQL `NoChainSync` table to Task 431's existing address integration
suite. Reuse its public allocation, lookup, concurrency, and reopen helpers
without altering its original cases.

## Verification

- Run numbered `NoChainSync` allocation, lookup, concurrency, and reopen cases
  unchanged on SQLite and PostgreSQL.
- Prove the new fixtures add no fabricated derivation path, public API, or
  direct Store assertion.

## Definition of done

- Numbered `NoChainSync` address results have focused public coverage without
  changing Task 431's existing contract.
- The modern-kvdb structural exclusion and SQL result metadata are truthful.
- No new method, result type, component, arbitrary file, or duplicated Task
  431 case is introduced.

[issue1250]: https://github.com/btcsuite/btcwallet/issues/1250
[t395]: 395-key-only-locator-allocation.md
[t431]: 431-address-allocation-lookup-itests.md
