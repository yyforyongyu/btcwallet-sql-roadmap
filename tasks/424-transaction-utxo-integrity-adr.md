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
- Keep Task 351's tenancy, Task 409's optional input representation, Task 410's
  concurrent mutation behavior, Task 411's terminal-parent admission, Task
  413's exact spend relation, Task 414's amount backstop, and Task 426's output
  validation independently scoped. Remove architectural-only hard edges.
- If current code already satisfies a proposed outcome and no necessary
  change remains, identify that task for supersession rather than creating
  implementation work to adopt a preferred structure.
- Keep this work deferred. Tasks 373–376 validate and import supported legacy
  facts directly; they do not require a transaction-graph redesign.
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
