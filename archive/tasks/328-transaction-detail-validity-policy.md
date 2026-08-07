# Task 328 — Define transaction-detail validity policy

## Status

Deferred, not scheduled. This does not block PRs #1272, #1291, or #1292.

## Problem

`db.GetTxDetail` and `db.ListTxDetails` return `TxDetailInfo.Status`, including
pending, published, failed, replaced, and orphaned states. The new public
`wallet.GetTx` and `wallet.ListTxns` conversion builds `wallet.TxDetail`
without carrying that state or filtering terminal invalid states. A failed,
replaced, or orphaned row can therefore look like an ordinary unconfirmed
transaction to a library caller.

The delivered stack deliberately does not add `wallet.TxStatus` or
`TxDetail.Status`; that was outside its minimal Store-routing scope. Removing
the inaccurate comment in #1291 is not a decision that the status can be
ignored forever.

Task 1000 BWR-001 is the read-only reference for a wallet-owned status type.
Do not duplicate or edit that broader plan here. This task owns only the
transaction-reader behavior and compatibility decision.

## Suggested fix

Use a later wallet-library PR to choose and document one explicit contract:

1. Surface a wallet-owned status on `TxDetail`, converting from the internal
   DB enum at the public boundary; or
2. Filter failed, replaced, and orphaned rows when the public API is intended
   to preserve the legacy visible-transaction contract.

Before choosing, compare the deprecated reader's observable behavior and the
needs of external library consumers such as lnd. Do not expose
`wallet/internal/db.TxStatus` through a public type.

Add table-driven coverage for `GetTx` and `ListTxns` across every status and
prove the chosen behavior for both kvdb and SQLite. Do not add a daemon itest
for this policy.

## Acceptance criteria

- `GetTx` and `ListTxns` document the same explicit status policy.
- Failed, replaced, and orphaned rows cannot silently masquerade as ordinary
  pending transactions.
- Public APIs expose no `wallet/internal/db` type.
- Focused tests cover pending, published, failed, replaced, and orphaned
  records for kvdb and SQLite.
- The work lands outside the three delivered SQL-wallet PRs.
