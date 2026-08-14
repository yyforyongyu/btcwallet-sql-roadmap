# Task 413 — constrain SQL UTXOs to exact spending inputs

Component: **SQL UTXO and normalized-input schema**.

Type: **Bug fix**.

Dependencies: **[Task 351][t351], [Task 409][t409], and [Task 410][t410]**.

## Rationale

Replace the loose `(spent_by_tx_id,
spent_input_index)` pointer with a referenceable normalized-input identity that
also proves the input spends this UTXO's exact creating outpoint. Task 424's
accepted design is consumed transitively through both dependencies.

## Details

Define
late-parent enrichment before allowing the pointer. Preflight mismatched,
missing, duplicate, cross-wallet, and terminal-spender pointers with
deterministic refusal or repair. Run every late-parent enrichment under Task
410 graph admission and, in the same SQL transaction that installs the spend
pointer, require the referenced spender/input row to retain an active status.
A terminal transaction cannot acquire or regain UTXO spend ownership.

Exclusions: general UTXO tenancy, amount bounds, graph traversal,
invalidation, replacement policy, and application-side monetary checks.

## Suggested fix

Make normalized inputs expose one unique
wallet/spender/input/previous-outpoint key and reference it from each non-null
UTXO spend relation. Make late-parent enrichment an explicit Task 410 graph
mutation whose conditional write joins the same spender/input identity and
active transaction status in one SQL transaction.

## Verification

Both dialects reject an existing input index that spends a
different outpoint, accept exact and late-parent vectors, and refuse/repair
every populated mismatch before the new constraint becomes authoritative.
Barrier tests race enrichment with invalidation and terminal transition;
neither ordering lets a terminal spender retain or regain the pointer, and
the status predicate is evaluated in the same SQL transaction as enrichment.

## Definition of done

Every non-null UTXO spend pointer declaratively names
the exact normalized transaction input that spends its creating outpoint, and
the pointer can be installed only while that same spender is active under Task
410 admission.

[t351]: 351-sql-utxo-integrity.md
[t409]: 409-transaction-input-graph.md
[t410]: 410-transaction-graph-admission.md
