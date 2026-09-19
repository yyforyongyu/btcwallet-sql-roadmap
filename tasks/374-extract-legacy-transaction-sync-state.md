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
  arithmetic, and reconstruct graph and balance facts from primary records.
- Compare independently stored indexes and pointers with that reconstruction.
  Refuse contradictory or unrepresentable history instead of selecting one
  version, dropping it, or substituting a rescan.
- Bound allocations and publish no partial snapshot on error or cancellation.
- Keep implementation in Wallet-owned migration code and legacy packages
  unchanged.
- Exclude destination mutation, public APIs, transaction-policy redesign, and
  normalization of unsupported history.

## Suggested fix

Build one private normalized graph from primary legacy records, derive its
expected indexes and aggregates, then compare the stored secondary facts. The
internal graph representation is not part of the contract.

## Verification

- Representative mined, unmined, spent, conflicted, labeled, leased, and
  reorg-retained histories preserve exact transaction and sync facts.
- Corrupt identities, indexes, pointers, amounts, statuses, and usedness fail
  before destination access.
- Unsupported multiple-spender or lease shapes are refused without source
  mutation.
- Cancellation and resource limits retain no partial published snapshot.

## Definition of done

A supported legacy history yields one internally consistent normalized
transaction, UTXO, lease, block, and synchronization snapshot without source
or destination mutation.

[t335]: 335-kvdb-to-sql-migration.md
[t473]: 473-extract-legacy-address-script-state.md
