# Task 411 — reject children of terminal transactions

Component: **TxStore activation preflight and direct/batch ingestion**.

Dependencies: **[Task 339][t339], [Task 409][t409], and [Task 410][t410]**.

## Rationale

Inside the same Store transaction and graph
admission that writes normalized inputs, reject any direct, sync-batch, or
scan-batch transaction whose input names a retained terminal parent. Close the
upgrade interval after Task 409 by refusing activation while an existing
active child still names a retained terminal parent.

## Details

Before enabling the new admission rule, acquire Task 410's graph admission and
scan every existing active transaction against its normalized parents. Abort
activation atomically with exact wallet, child, input, and terminal-parent
diagnostics if any violation exists; do not silently delete history or choose
a replacement status. After a clean preflight, reject new violations before
credits, debits, labels, or normalized edges become visible. Define idempotent
replay of the same active transaction without legitimizing a terminal-parent
branch.

Exclusions: public invalidation, descendant traversal, publication,
replacement policy, and raw transaction backfill.

## Suggested fix

Add one activation preflight over existing normalized edges under graph
admission, then query the status of all normalized parent hashes inside the
same admitted Store transaction before inserting the child row, edge rows, or
wallet effects. Share the parent-status predicate across preflight, direct,
and batch paths so cutover and future admission enforce the same invariant.

## Verification

SQLite/PostgreSQL fixtures inserted after Task 409 but before this task cause
activation to fail atomically with exact diagnostics, while a clean fixture
enables admission. Every ingestion path then rejects terminal-parent children
with zero partial effects; active-parent and unrelated-input cases succeed;
replay is deterministic.

## Definition of done

No active descendant of a retained terminal transaction survives activation,
and no maintained Store ingestion path can create a new one afterward.

[t339]: 339-transaction-visibility.md
[t409]: 409-transaction-input-graph.md
[t410]: 410-transaction-graph-admission.md
