# Task 356 — defer change-script allocation until viable

Component: **TxCreator, PsbtManager, and Wallet-owned address storage**.

Type: **Bug fix**.

Dependencies: **[Task 344][t344], [Task 355][t355], [Task 357][t357],
[Task 404][t404], and [Task 467][t467]**.

## Rationale

`TxCreator.CreateTransaction` and `PsbtManager.FundPsbt` can derive and commit
a change address before authoring proves that positive, non-dust change will
appear in the returned transaction. Failed, no-change, and retried requests can
therefore consume an address that the caller never receives.

## Details

- Prepare the exact candidate needed by the existing authoring call without
  durably advancing its address branch.
- Discard the candidate when authoring fails or produces no viable change.
- Before publication or lease effects, atomically commit the exact viable
  candidate using Task 344's SQL allocation boundary or modern kvdb's existing
  count-one transaction.
- If another allocation moved the expected cursor, discard the authored result
  and retry with the caller's context. No other failure is retryable.
- Reconcile an ambiguous commit from exact backend facts. Continue only when
  the candidate is fully committed, return the original error for clean
  absence, and return Task 467's indeterminate identity for every other state.
- Validate only the descriptor facts mechanically required to prepare and
  commit the exact candidate. Do not add a broader descriptor-policy matrix
  without a separate reproduced public bug.
- Exclude changes to `wallet/txauthor`, `waddrmgr`, or `walletdb`; copied
  viability arithmetic; watched-change delivery; PSBT metadata; and leasing
  policy.

## Suggested fix

Use a private candidate passed through the existing change-source boundary,
then commit it only after authoring selects it. The preview, cache, and retry
mechanics may differ by backend as long as they preserve the public invariants.

## Verification

- Exercise both public callers on kvdb, SQLite, and PostgreSQL for authoring
  failure, no change, dust change, viable change, cursor conflict, and
  ambiguous commit.
- Prove only viable change advances the cursor and that the committed script
  exactly matches the returned transaction.
- Prove a cursor conflict retries before publication or leases, while an
  ambiguous result is never retried and is classified as exact, absent, or
  indeterminate.
- Cover only descriptor validation required by the tested candidates.

## Definition of done

Maintained transaction creation and PSBT funding consume no change address
unless viable change uses that exact address, with no unrelated descriptor
policy or excluded-package change.

[t344]: 344-bulk-address-allocation.md
[t355]: ../archive/tasks/355-transaction-size-accounting.md
[t357]: 357-transaction-amount-validation.md
[t404]: 404-transaction-input-amounts.md
[t467]: ../archive/tasks/467-indeterminate-commit-identity.md
