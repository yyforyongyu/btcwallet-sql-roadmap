# Task 351 — enforce SQL UTXO integrity by constraint

Component: **UtxoManager and SQL Store**.

Dependencies: **[Task 340][t340] and [Task 349][t349], plus an accepted ADR
0006 amendment or supersession covering declarative UTXO ownership and
consistency**.

## Rationale

`utxos` lacks `wallet_id` and relies on backend-specific triggers for tenancy.
Its spend pointer can name an existing input that actually spends a different
outpoint, and wire indexes do not uniformly cover the unsigned 32-bit range.

## Details

- Add `wallet_id` and composite wallet/transaction/address foreign keys.
- Tie a spend pointer to the UTXO's exact creating outpoint, not only an input
  index.
- Use a concrete non-partial unique parent key that PostgreSQL can reference,
  such as a surrogate transaction-output identity or a full unique composite.
- Give leases a composite `(wallet_id, utxo_id)` reference.
- Use explicit `RESTRICT` plus application-owned pointer clearing.
- Make wire indexes unsigned-32-bit-capable on both SQL dialects.
- Enforce every UTXO amount within `0..MaxSatoshi`. Validate each output and its
  checked transaction aggregate before direct or batch insertion on both
  dialects.
- Before adding constraints, detect orphaned, duplicate, cross-wallet, and
  mismatched rows plus every wire index outside `0..MaxUint32`, and apply a
  documented deterministic repair or refusal rule. Include existing negative,
  over-`MaxSatoshi`, and overflowing-aggregate amount rows in that preflight.
- Amend or supersede ADR 0006 before replacing its deliberate derived ownership
  and trigger-based consistency design.

## Suggested fix

Extend normalized transaction inputs from [Task 349][t349] with a coherent local parent
identity, clean or reject invalid existing rows, then replace consistency
triggers with referenceable unique keys and composite foreign keys. Define
late-parent enrichment before allowing the spend pointer.

## Verification

Cross-wallet references, mismatched spend outpoints, and invalid wire indexes
are rejected declaratively on SQLite and PostgreSQL, including late-parent and
lease vectors. Existing over-`uint32` SQLite rows are deterministically refused
or repaired before constraints. Invalid individual and aggregate amounts fail
before mutation, existing invalid amounts are refused or repaired, and the
accepted ADR matches the implemented ownership model.

## Definition of done

- SQLite and PostgreSQL enforce wallet tenancy, exact spend-outpoint identity,
  lease ownership, wire-index range, and monetary bounds declaratively.
- Existing invalid rows follow the documented deterministic repair or refusal
  rule before constraints become authoritative.
- The accepted ADR and maintained data-model documentation match the final
  ownership and pointer semantics.

[t340]: 340-durable-database-identity.md
[t349]: 349-transaction-invalidation.md
