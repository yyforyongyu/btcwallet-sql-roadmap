# Task 424 — define transaction and UTXO integrity ownership

Component: **ADR 0006 transaction and UTXO integrity contract**.

Type: **Design**.

Dependencies: **None**.

## Rationale

Draft ADR 0006 mixes public correctness requirements with proposed normalized
inputs, schema constraints, and mutation coordination. Existing SQL
invalidation already reconstructs complete descendant relationships from raw
transactions. A different representation is not, by itself, a missing Wallet
behavior or a prerequisite for legacy conversion.

## Details

- Accept or supersede ADR 0006 against the maintained Wallet ingestion,
  TxReader, UtxoManager, and transaction-invalidation paths. For each proposed
  bug fix, record the current entry point, reachable input or interleaving,
  existing protection, and wrong public result.
- Distinguish a reproduced failure from defensive schema hardening and an
  optional representation refactor. Unsupported direct SQL corruption alone
  does not establish a public API bug.
- Specify wallet tenancy, exact spending outpoints, late-parent arrival,
  terminal-transaction behavior, and amount bounds without assuming a new
  input table or graph coordinator.
- Compare existing raw-transaction reads, transaction boundaries, predicates,
  and constraints with the smallest proposed correction. Require a concrete
  consumer need before accepting normalized inputs or extra coordination.
- Record the 2026-09-18 pruning decision: preserve existing tenancy triggers,
  raw-transaction graph reads, and exact-spend predicates. Tasks 351, 409,
  413, and 414 are superseded; no new representation or optional amount CHECK
  is required. Keep Task 410's demonstrated concurrency concern, Task 411's
  non-owned terminal-parent gap, and Task 426's raw-output admission distinct.
  Task 426 can use the existing admission boundary without waiting for this
  ADR; Task 410 still requires its evidence and ordering decision.
- If current code already satisfies a proposed outcome and no necessary
  change remains, identify that task for supersession rather than creating
  implementation work to adopt a preferred structure.
- Prioritize Task 410's ordering decision required by Task 454. Tasks 373–376
  validate and import supported legacy facts directly; they do not require a
  transaction-graph redesign.
- Exclude implementation, new public APIs, replacement-policy redesign,
  generic locks, and cross-process fencing.

## Suggested fix

Amend the existing ADR with concise issue/cause/evidence/outcome entries and
the smallest justified ownership decisions. Reuse current behavior whenever
it already satisfies the accepted invariant.

## Verification

- Trace direct and batch ingestion, invalidation, rollback, deletion, and
  late-parent arrival on SQLite and PostgreSQL.
- Cite current source and existing tests for satisfied invariants. For a
  proposed correction, give a reproducible public-contract case or a concrete
  source-level interleaving; label any unvalidated concern explicitly.
- Confirm each retained implementation task has one necessary outcome and
  that no optional representation change gates a behavioral fix.
- Check ADR status and links. This design task adds no production code.

## Definition of done

An accepted ADR distinguishes proven fixes, defensive hardening, and optional
refactors, with evidence and minimal ownership boundaries. Its task decisions
do not turn preferred architecture into a migration prerequisite.
