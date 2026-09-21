# Task 356 — defer change-script allocation until viable

Component: **TxCreator, PsbtManager, and Wallet-owned address storage**.

Type: **Bug fix**.

Dependencies: **[Task 355][t355], [Task 357][t357], [Task 404][t404],
and [Task 467][t467]**.

## Rationale

`TxCreator.CreateTransaction` and `PsbtManager.FundPsbt` can commit a change
address before authoring proves that positive, non-dust change is needed.
Failed, zero-change, and dust-change attempts can consume unused addresses.

## Details

- Determine viable change using the required script properties before durable
  allocation. Do not require previewing or reserving an actual address.
- Once change is viable, allocate once and use that committed script in the
  returned transaction. Preserve fee, dust, and concurrent-allocation
  correctness through the existing Wallet-owned authoring boundary.
- Reuse existing backend allocation; do not require Task 344's SQL batch API
  solely for implementation reuse or route kvdb through it.
- An ambiguous allocation commit returns `ErrIndeterminateCommit` and no
  result, without automatic retry or claiming rollback.
- Keep watch delivery in [Task 474][t474] and lease policy in Tasks 405/406.
  A later wrapper failure does not undo an already committed address.
- Exclude changes to `wallet/txauthor`, `waddrmgr`, or `walletdb`, copied
  fee/dust arithmetic, cursor reservation/retry protocols, and reconciliation
  machinery.

## Suggested fix

Separate the script properties needed for authoring from durable allocation
at the existing Wallet-owned change-source boundary. Choose the smallest
private implementation that allocates only after viable change is known.

## Verification

- Exercise both callers on kvdb, SQLite, and PostgreSQL for authoring failure,
  zero change, dust change, and viable change.
- Only viable change advances the address cursor, and the returned script
  matches the committed address without altering fee or dust decisions.
- Concurrent calls do not reuse a change address. Ambiguous allocation returns
  the shared error identity without a result or retry.

## Definition of done

Both public callers avoid durable allocation for zero/dust change and use
one committed address for viable change, without redesigning allocation,
watching, or leasing.

[t355]: ../archive/tasks/355-transaction-size-accounting.md
[t357]: ../archive/tasks/357-transaction-amount-validation.md
[t404]: ../archive/tasks/404-transaction-input-amounts.md
[t467]: ../archive/tasks/467-indeterminate-commit-identity.md
[t474]: 474-authored-change-watch.md
