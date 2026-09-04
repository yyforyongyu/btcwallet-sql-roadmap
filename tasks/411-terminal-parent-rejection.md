# Task 411 — reject children of terminal transactions

Component: **Wallet-owned SQL transaction ingestion**.

Type: **Bug fix**.

Dependencies: **[Task 339][t339] and [Task 410][t410]**.

## Rationale

A transaction learned through maintained Wallet ingestion must not remain
active when one of its retained parents is terminal. Otherwise TxReader can
report an active branch and UtxoManager can retain effects from a parent that
the Wallet has invalidated. Existing and newly arriving children need the
same admission rule, independent of how input relationships are represented.

## Details

- Under Task 410's mutation-ordering contract, reject a new child whose input
  references a retained terminal parent before any Wallet effect is committed.
- Validate existing active transactions before enabling the rule. Refuse an
  inconsistent state with exact Wallet, child, input, and parent diagnostics;
  do not silently delete history or select a replacement status.
- Cover maintained direct and batch ingestion and preserve deterministic
  replay without legitimizing a terminal-parent branch.
- Read parent hashes from existing raw transaction/input data. Task 409's
  optional representation is not a prerequisite; do not require normalized
  rows, a graph-admission handle, or a new coordinator.
- Exclude public invalidation, descendant traversal, publication policy,
  replacement policy, and an independent input-graph migration.

## Suggested fix

Reuse the existing decoded inputs and parent-status checks inside the narrow
mutation boundary. Share the accepted predicate between existing-state
validation and future ingestion without prescribing private helper shapes.

## Verification

- On SQLite and PostgreSQL, refuse an existing active child of a terminal
  parent before partial activation and admit a clean state.
- Through maintained ingestion, reject terminal-parent children with no
  partial public transaction or UTXO effect. Accept active-parent and unrelated
  inputs and preserve idempotent replay.
- Race the parent transition with child admission under Task 410 and prove
  neither ordering leaves an active terminal-parent branch.

## Definition of done

No active child of a retained terminal transaction survives activation or
maintained ingestion, without depending on a new input representation.

[t339]: ../archive/tasks/339-transaction-visibility.md
[t410]: 410-transaction-graph-admission.md
