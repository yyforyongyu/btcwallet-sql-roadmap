# Task 374 — extract legacy transaction and sync state

Component: **Legacy migration transaction and sync extraction**.

Type: **Migration**.

Dependencies: **[Task 473][t473]**.

## Rationale

Legacy transaction state is spread across primary records and mined, unmined,
credit, debit, lease, and synchronization indexes. The modern kvdb projection
does not expose enough information to prove that an SQL import is complete and
internally consistent.

## Details

- Read only [Task 335][t335]'s immutable source and join ownership through Task
  473's normalized address identities.
- Extract raw transactions, block membership, status, labels, credits, debits,
  UTXOs, spends, supported conflict facts, active leases, birthday, sync tip,
  and the block history required to interpret them.
- Decode each transaction, verify its identity, use checked monetary
  arithmetic, and derive the graph and balance facts required by the import.
- Validate indexes and pointers that supply or contradict imported facts.
  Refuse contradictory or unrepresentable history instead of selecting one
  version, dropping it, or substituting a rescan. Do not audit unrelated
  legacy indexes or build a replacement transaction-graph subsystem.
- Bound allocations and publish no partial snapshot on error or cancellation.
- Keep implementation in Wallet-owned migration code and legacy packages
  unchanged.
- Exclude destination mutation, public APIs, transaction-policy redesign, and
  normalization of unsupported history.

## Suggested fix

Decode the required legacy records and cross-check their ownership, spending,
and balance relationships using existing readers. Private intermediate shapes
remain flexible; no standalone graph or general database auditor is required.

## Verification

- Representative mined, unmined, spent, conflicted, labeled, leased, and
  reorg-retained histories preserve exact transaction and sync facts.
- Corrupt imported identities, indexes, pointers, amounts, statuses, and
  usedness fail before destination access.
- Unsupported multiple-spender or lease shapes are refused without source
  mutation.
- Cancellation and resource limits retain no partial published snapshot.

## Definition of done

A supported legacy history yields one internally consistent normalized
transaction, UTXO, lease, block, and synchronization snapshot without source
or destination mutation.

[t335]: 335-kvdb-to-sql-migration.md
[t473]: 473-extract-legacy-address-script-state.md
