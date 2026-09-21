# Task 389 — report uncertain account commits truthfully

Component: **AccountManager SQL account-creation results**.

Type: **Bug fix**.

Dependencies: **[Task 343][t343], [Task 390][t390], and [Task 467][t467]**.

## Rationale

SQL `NewAccount` already maps an ambiguous commit to
`ErrIndeterminateCommit` and waits for admitted work. `ImportAccount`
needs the same result handling: an ordinary failure or caller cancellation
must not imply rollback when an account may have committed.

## Details

- Map an ambiguous SQL `ImportAccount` commit to the existing wallet-owned
  `ErrIndeterminateCommit`, returning no account.
- After mutation admission, await and deliver the write result; caller
  cancellation must not hide commit uncertainty. Retain cancellation before
  admission and preserve `NewAccount`'s existing behavior.
- Keep existing transaction and concurrency protections. Do not retry an
  uncertain write or add readback, retained-ciphertext comparison, or extra
  serialization solely to classify the outcome.
- Exclude genesis, path-bound account shapes, cursor redesign, and public API
  additions.

## Suggested fix

Reuse the existing `NewAccount` error mapping and admitted-result handling
in `ImportAccount`, without adding a reconciliation layer.

## Verification

- On SQLite and PostgreSQL, inject an ambiguous import commit: the result
  matches `ErrIndeterminateCommit`, contains no account, and is not retried.
- Race cancellation before and after admission. Only the former aborts
  without a write; the latter delivers the actual write result.
- Preserve successful imports, definite failures, and existing `NewAccount`
  ambiguity/cancellation results.

## Definition of done

SQL account creation reports uncertain writes conservatively, and admitted
import results cannot be replaced by caller cancellation.

[t343]: ../archive/tasks/343-exact-account-creation.md
[t390]: ../archive/tasks/390-custom-scope-account-creation.md
[t467]: ../archive/tasks/467-indeterminate-commit-identity.md
