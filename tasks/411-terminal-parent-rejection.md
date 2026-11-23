# Task 411 — reject children of terminal transactions

Component: **Wallet-owned SQL transaction ingestion**.

Type: **Bug fix**.

Dependencies: **[Task 339][t339] and [Task 410][t410]**.

## Rationale

[PR #1512][pr1512] rejects new unconfirmed children of known terminal
parents on both SQL backends, including when the referenced output carries
no Wallet credit. Its [retained-input validation][validate] allows a
genuine confirming block subject to the confirmed-conflict check. The
existing [parent-without-credit regression][regression] exercises both
outcomes.

Startup still needs to refuse stored active unconfirmed children of known
terminal parents before publishing the Wallet. Otherwise TxReader and
UtxoManager can expose effects from an invalid unconfirmed branch. The
regression also stores a block-confirmed child as Published while its
parent remains Failed. That history must reopen.

## Details

- Before Wallet activation, refuse a stored active unconfirmed child of a
  known terminal parent. Report the Wallet, child, input and parent, with
  no partial activation, deletion or status repair.
- Honor a stored confirming block for an otherwise valid child even if its
  known parent remains terminal. Preserve the confirmed-conflict guard.
- Preserve delivered admission on direct and batch ingestion, no partial
  Wallet effects on rejection, and deterministic replay under Task 410's
  mutation ordering.
- Leave confirmation of an already retained terminal child to the separate
  transaction cutover. The confirming-block admission case and reuse of an
  existing terminal row are distinct.
- Exclude public invalidation, descendant traversal, publication policy,
  replacement policy, and an independent input-graph migration.

## Suggested fix

Use the retained inputs and existing transaction status checks to
validate stored history before activation. Reuse the delivered admission
rule where the startup contract calls for the same predicate.

## Verification

- On SQLite and PostgreSQL, reopen otherwise valid history with a
  block-confirmed child of a known terminal parent. Refuse stored history
  with an active unconfirmed child of a known terminal parent, with the
  required diagnostics and before Wallet activation.
- Reuse the existing ingestion regression for unconfirmed rejection without
  a transaction or UTXO effect and acceptance with a genuine confirming
  block. Preserve valid-parent, unrelated-input and idempotent replay
  behavior.
- Where existing evidence does not cover the parent transition racing an
  unconfirmed child, prove neither ordering under Task 410 leaves an
  active unconfirmed child of a known terminal parent.

## Definition of done

SQLite and PostgreSQL reopen valid confirmed history and refuse
inconsistent unconfirmed history with actionable diagnostics before
Wallet activation. Delivered ingestion and mutation ordering remain covered.

[t339]: ../archive/tasks/339-transaction-visibility.md
[t410]: ../archive/tasks/410-transaction-graph-admission.md
[pr1512]: https://github.com/btcsuite/btcwallet/pull/1512
[validate]: https://github.com/btcsuite/btcwallet/blob/70f5629b6f5a75b29582eac9251c804d61bd7b20/wallet/internal/db/txstore_common.go#L1197
[regression]: https://github.com/btcsuite/btcwallet/blob/70f5629b6f5a75b29582eac9251c804d61bd7b20/wallet/internal/db/itest/txstore_utxostore_test.go#L456
