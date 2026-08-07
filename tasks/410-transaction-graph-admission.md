# Task 410 — coordinate transaction graph mutations

Component: **Wallet/Store transaction graph admission**.

Dependencies: **[Task 334][t334] and [Task 409][t409]**.

## Rationale

Provide one operation-specific, wallet-scoped
admission contract for transaction status/edge changes.

## Details

Route CreateTx,
UpdateTx graph-state transitions, ApplyTxBatch, ApplyScanBatch, DeleteTx,
invalidation, wallet rewind, and rollback through it.
[Task 372][t372]'s database-wide rollback coordinator acquires affected
wallets in stable wallet-ID order.

Exclusions: a generic Store write lock, holding a database transaction
across network calls, cross-process/Manager fencing, invalidation semantics,
terminal-parent policy, and publication behavior.

## Suggested fix

Add a narrow per-wallet transaction-graph coordinator at
the Wallet/Manager operation boundary and pass an admission handle into graph
mutations; let [Task 372][t372] acquire multiple handles in sorted wallet-ID
order.

## Verification

Barrier tests cover each named graph mutation, same-wallet
operations serialize, unrelated wallets remain independent, and multi-wallet
rollback acquires stable order without inversion.

## Definition of done

Every graph mutation has one visible admission point
that later invalidation and publication tasks can consume; no general write
serialization or cross-process claim is added.

[t334]: 334-shared-chain-state.md
[t409]: 409-transaction-input-graph.md
[t372]: 372-multi-wallet-rollback.md
