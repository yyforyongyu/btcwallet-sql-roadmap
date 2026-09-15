# Task 461 — cover public address batch contracts

Component: **AddressManager batch integration tests**.

Type: **Test**.

Dependencies: **[Task 343][t343], [Task 344][t344], and
[Task 468][t468]**.

## Rationale

[Issue #1250][issue1250] requires AddressManager integration coverage.
`NewBulkAddresses` forms one independently changeable public contract,
separate from count-one allocation and lookup behavior in Task 431.

## Details

- Exercise Task 344's final `NewBulkAddresses` contract through imported
  `wallet` APIs on real SQLite and PostgreSQL Wallets without defining its
  implementation shape in this test task. A successful call returns the final
  usable batch; delivery bookkeeping remains internal.
- Before allocation, derive the expected children independently from the
  created account's public XPub, persisted schema, selected branch, and known
  initial branch index. Use those independently derived values to prove exact
  returned order, account, branch, child index, address, public key, and
  metadata rather than constructing expectations from the returned batch.
- Prove durable point/list visibility after successful delivery and no usable
  partial returned result. Reopen the Wallet after completion and require the
  independently expected batch and branch progress to remain durable.
- Cover counts 1 and `MaxBulkAddressCount`. Prove zero and
  `MaxBulkAddressCount+1` match `ErrInvalidParam` through `errors.Is` before
  mutation on ordinary SQL accounts, modern kvdb, and a `NoChainSync` SQL
  account, so request validation demonstrably precedes capability checks.
- Keep commit-before-cancel, failed live registration, and startup watch
  reconstruction in Tasks 344 and 402's colocated tests. Do not add a public
  delivery handle, result wrapper, recovery API, or real-backend fault seam in
  this integration task.
- Add a modern-kvdb public contract proving batch allocation matches Task
  378's canonical `ErrAccountOperationUnsupported` through `errors.Is` before
  mutation. Kvdb cannot represent Task 344's SQL batch mutation, so keep it
  outside the SQL success and registration-failure cases rather than skipping
  it inside a shared contract.
- On SQLite and PostgreSQL, create a `NoChainSync=true` account through Task
  468's activation of the public account request. A valid batch request must
  match Task 378's
  canonical `ErrAccountOperationUnsupported`, with Task 392's precedence,
  before any publicly observable address result or branch progression on both
  dialects. Task 344's colocated injected-boundary tests own the unexposed
  no-registration postcondition.
- Keep ambiguous-commit and terminal-exhaustion fault injection in Task 344's
  colocated tests rather than adding a real-backend fixture seam here.
- Exclude count-one allocation, oldest-unused selection, import/script APIs,
  direct Store inspection, RPC, deprecated helpers, and unrelated production
  changes.
- Extend an existing address integration-test file when it can own these
  cases. Add no arbitrary helper or standalone test file; if the repository
  has no suitable file, justify one repository-conventional itest file in the
  implementation PR.
- Document only non-obvious test contracts, rationales, invariants, and
  consequences. Keep Arrange, Act, and Assert structure clear without
  mandatory section comments that restate the test code.
- Interface mocks, when needed, embed `mock.Mock`. Set `On(...).Return(...)`
  expectations during Arrange with the narrowest correct cardinality (`Once()`
  for one required call). During Assert, call `AssertExpectations(t)` directly;
  do not duplicate call assertions or wrap the result in `require.True`.

## Suggested fix

Extend the existing AddressManager integration suite with one SQL batch table
covering both dialects and one modern-kvdb unsupported case. Derive expected
children from public account XPub, schema, branch, and initial state before the
allocation, then reuse public point/list reads and a durable reopen to prove
the returned batch and cursor state.

## Verification

- Run the focused batch cases on SQLite and PostgreSQL.
- Verify count boundaries, ordered results, durable reopen, and complete
  public state after each successful call.
- Run valid-count unsupported cases on modern kvdb and `NoChainSync` SQLite
  and PostgreSQL accounts, and prove through public lookup, list, and reopen
  results that they expose no address change. On modern kvdb only, use a
  follow-up count-one allocation to prove the rejected batch did not advance
  its branch. `NewAddress` remains unsupported on `NoChainSync` fixtures;
  Task 395's separate `AllocateNextKey` operation is outside this batch task.
- Run zero and 101 on modern kvdb and both `NoChainSync` SQL fixtures, require
  `ErrInvalidParam`, and prove the same publicly observable postcondition.

## Definition of done

- The exported batch result has matching public coverage on SQLite and
  PostgreSQL.
- Public count bounds match `ErrInvalidParam` before capability checks, and
  valid-count modern-kvdb and `NoChainSync` SQL requests return the canonical
  unsupported identity before mutation.
- Tests use imported `wallet` APIs and real maintained backends without direct
  Store assertions or unrelated production changes.

[issue1250]: https://github.com/btcsuite/btcwallet/issues/1250
[t343]: 343-exact-account-creation.md
[t344]: 344-bulk-address-allocation.md
[t468]: ../archive/tasks/468-activate-no-chain-sync-account-creation.md
