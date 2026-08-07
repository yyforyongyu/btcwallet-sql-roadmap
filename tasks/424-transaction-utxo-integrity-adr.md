# Task 424 — define transaction and UTXO integrity ownership

Component: **ADR 0006 transaction and UTXO integrity contract**.

Dependencies: **None**.

## Rationale

Draft ADR 0006 spans wallet tenancy, normalized transaction inputs, spend
identity, late-parent handling, and deletion ownership. The SQL schema tasks
cannot independently choose compatible constraints or mutation order until one
accepted design assigns each integrity rule to a single owner.

## Details

- Accept or supersede ADR 0006 with one backend-neutral SQL contract for
  wallet-scoped UTXOs, leases, transactions, inputs, and addresses.
- Define one normalized row for every transaction input, including external
  previous outpoints, and distinguish graph identity from wallet ownership.
- Define how a non-null UTXO spend relation names the exact normalized input
  that spends its creating outpoint rather than only a transaction and index.
- Define late-parent enrichment for a child or spend observed before its
  wallet-owned parent output, including the only operation allowed to attach
  the later UTXO identity.
- Assign mutation, invalidation, rollback, and deletion ownership so graph
  edges and spend relations cannot be removed or rewritten by competing paths.
- Separate design from implementation. Task 351 owns wallet tenancy and its
  UTXO-carried wire indexes; Task 409 owns normalized input rows and their
  input-position and previous-output-index bounds; Task 413 owns exact spend
  identity; Task 414 owns individual persisted UTXO amount bounds; and Task 426
  independently owns persisted SQL transaction output-set admission.
- Exclude schema migrations, generated queries, public invalidation APIs,
  replacement-policy redesign, checked arithmetic implementation, generic
  write locks, and cross-process fencing.

## Suggested fix

Finalize ADR 0006 as a constraint and mutation-ownership record. Walk each
ingestion, late-parent, invalidation, rollback, and deletion path, assign one
authoritative row or relation to each fact, and link the four implementation
tasks without prescribing dialect-specific SQL.

## Verification

- Review the accepted record against both SQL schemas, transaction ingestion,
  UTXO creation and spending, late-parent arrival, invalidation, rollback, and
  deletion call paths.
- Trace direct and batch ingestion with internal, external, late-parent,
  conflicting, terminal, and cross-wallet inputs and require one unambiguous
  owner for every durable relation.
- Confirm SQLite and PostgreSQL can implement the same contract and that Tasks
  351, 409, 413, 414, and 426 each own one non-overlapping implementation
  boundary.
- Validate ADR status, index, and task links; this task runs no btcwallet code
  tests.

## Definition of done

- An accepted and merged ADR defines wallet tenancy, normalized input edges,
  exact spend identity, late-parent enrichment, and mutation/deletion
  ownership.
- Tasks 351, 409, and 426 cite this accepted task directly, while Tasks 413 and
  414 consume only their implementation prerequisites.
- The decision adds no schema migration, generated code, public API, generic
  lock, or cross-process guarantee.
