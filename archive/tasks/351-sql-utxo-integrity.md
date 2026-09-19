# Task 351 — bind SQL UTXOs to wallet tenancy

Component: **SQL UTXO and lease schema**.

Type: **Bug fix**.

Dependencies: **[Task 340][t340] and [Task 424][t424]**.

> **Superseded 2026-09-18:** Keep the existing wallet-consistency triggers in
> migrations 000008 and 000010 and wallet-scoped writers. At sql-wallet
> 2ae306dea8c5, no reachable public tenancy failure justifies replacing them
> with duplicate wallet IDs/composite foreign keys or expanding wire-index
> storage. This optional representation change is eliminated; Task 424 records
> the retained integrity contract.

## Rationale

Add `wallet_id` to UTXOs; add concrete referenceable
unique keys and composite wallet/creating-transaction/address/spending-
transaction foreign keys; give leases a composite `(wallet_id, utxo_id)`
reference; use explicit RESTRICT plus application-owned pointer clearing; and
make every wire index introduced or retained by this UTXO migration cover
`0..MaxUint32` on both dialects. Task 409 separately owns any justified input
representation follow-up. Task 424 supplies the accepted ownership contract.

## Details

Preflight orphan, duplicate, cross-wallet, and out-of-range UTXO-carried rows
with an exact repair-or-refusal rule before replacing triggers.

Exclusions: exact previous-outpoint matching, amount bounds, transaction
input graph creation, invalidation behavior, and checked monetary arithmetic.

## Suggested fix

Migrate UTXOs to carry wallet ID, add the required composite
unique parent keys/FKs for transactions and addresses, then point leases at a
composite wallet/UTXO key and remove the consistency triggers after preflight.

## Verification

SQLite/PostgreSQL reject cross-wallet transaction, address,
spend, and lease references and invalid UTXO-carried wire indexes
declaratively; populated migrations deterministically repair or refuse every
preflight class; generated queries and maintained ADR/data-model text match.

## Definition of done

UTXO and lease tenancy plus UTXO-carried wire-index ranges are enforced by
composite constraints rather than backend triggers, with no normalized-input,
exact-spend, or amount work bundled in.

[t340]: 340-durable-database-identity.md
[t424]: ../../tasks/424-transaction-utxo-integrity-adr.md
