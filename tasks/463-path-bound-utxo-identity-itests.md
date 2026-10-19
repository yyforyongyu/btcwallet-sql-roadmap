# Task 463 — verify path-bound UTXO account identity

Component: **UtxoManager path-bound account integration tests**.

Type: **Test**.

Dependencies: **[Task 391][t391]**.

## Rationale

[Issue #1250][issue1250] requires UtxoManager integration coverage. Completed
Task 415 predates Task 391's representable numbered path-bound XPub account
shape, so no public test proves that its funded outputs retain the correct
account identity through SQL UTXO conversion.

## Details

- This tests the new runtime account shape from Task 391, not legacy snapshot
  conversion. Keep it in Phase 2 with that real dependency; it is not a
  migration gate. Tasks 373–376 own converted legacy account/UTXO equivalence.
- Exercise `ListUnspent` and `GetUtxo` only through the imported `wallet`
  package on real SQLite and PostgreSQL Wallets.
- Create a uniformly watch-only, path-bound XPub account with
  `NoChainSync=false`, allocate a managed public address through the maintained
  AddressManager API, fund it through the existing harness, and wait for the
  public Wallet view to recognize the output.
- Require both methods to report the path-bound account's public name and the
  same output metadata. Account-filtered `ListUnspent` must include the output
  for that account and exclude it for another account.
- Reopen the same Wallet and prove point lookup, account attribution, and
  account-filter behavior remain unchanged.
- Assert only the existing public UTXO contract. Task 391 may retain the real
  account number internally to produce the truthful public account identity;
  do not add an account-number field, Store query, or parallel public API for
  this test.
- Modern kvdb cannot represent Task 391's numbered path-bound XPub account, so
  exclude it structurally from this SQL-only fixture rather than adding a skip
  branch or synthetic backend behavior.
- Extend the existing `itest/utxo_manager_test.go` suite and reuse its funding
  and reopen infrastructure. Add no arbitrary helper, production file, or
  standalone test file; justify any unavoidable new file against the current
  repository layout in the implementation PR.
- Keep direct Store access, raw SQL assertions, leases, unrelated UTXO cases,
  RPC, deprecated APIs, and unrelated production changes out of scope. Fix
  only a production defect required by this public attribution contract and
  keep its regression in this task's implementation PR.

## Suggested fix

Extend the existing UtxoManager integration suite with one shared SQL case.
Build the expected account name and outpoint from public account, address, and
funding results, then compare `ListUnspent`, its account filter, and `GetUtxo`
before and after reopening the same durable wallet.

## Verification

- Run the focused path-bound UTXO case unchanged on SQLite and PostgreSQL.
- Prove list, account-filter, and point results agree on the funded outpoint and
  path-bound account name before and after reopen.
- Verify the change extends the existing suite and adds no public field, Store
  assertion, kvdb-only behavior, or unrelated UTXO coverage.

## Definition of done

- A funded path-bound XPub output has truthful and durable public account
  attribution through `ListUnspent` and `GetUtxo` on both SQL dialects.
- The test uses imported `wallet` APIs and real backends without direct Store
  inspection or a synthetic kvdb representation.
- No new component, public UTXO field, API, helper file, or standalone test
  file is introduced.

[issue1250]: https://github.com/btcsuite/btcwallet/issues/1250
[t391]: ../archive/tasks/391-path-bound-watch-only-xpub-accounts.md
