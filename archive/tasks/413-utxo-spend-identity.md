# Task 413 — validate exact UTXO spending inputs

Component: **Wallet-owned SQL UTXO spend assignment**.

Type: **Bug fix**.

Dependencies: **[Task 351][t351] and [Task 410][t410]**.

> **Superseded 2026-09-18:** At sql-wallet 2ae306dea8c5, markInputsSpent
> derives outpoints and input indices from the same decoded transaction;
> addChildInputSpends matches exact parent hash/index and refreshes active
> status. TestCreateTxReplayedCreditSkipsReplacedChildSnapshot covers stale
> replay. Eliminate duplicate identity machinery; the separate concurrent
> graph-ordering concern remains in Task 410.

## Rationale

The stored spender transaction and input index do not themselves prove that
the input references this UTXO's creating outpoint or that the spender remains
active. An incorrect assignment during Wallet ingestion or late-parent
enrichment makes UtxoManager hide an available output or attribute a spend to
the wrong transaction. Task 424 must identify the reachable writer and gaps in
its existing checks; an artificially corrupted SQL row is not enough evidence.

## Details

- Close only the accepted exact-spend assignment defect. Require the spender
  input to reference the UTXO's exact creating outpoint in the same Wallet.
- Keep the spender's active-state check and spend assignment atomic, including
  when a child arrives before its parent and enrichment installs the relation
  later. Task 410 owns ordering with competing graph mutations.
- Validate existing facts before any storage cutover. Refuse inconsistent
  identities without silently rewriting transaction meaning or spendability.
- Reuse existing raw transaction/input data and local predicates or
  constraints. Do not require Task 409's optional representation, a composite
  foreign-key shape, a new input table, or another admission handle.
- Preserve Task 351's tenancy. Exclude amount bounds, graph traversal,
  replacement policy, public APIs, and unrelated schema cleanup.
- If current writers already enforce the full invariant, use Task 424's
  evidence to supersede this fix instead of adding redundant infrastructure.

## Suggested fix

Validate the exact input and active spender at the existing spend-assignment
boundary. Choose a predicate, constraint, or other local correction according
to the reproduced failure; the storage representation is not the outcome.

## Verification

- Reproduce the accepted wrong-outpoint or stale-spender assignment and prove
  maintained public UTXO results remain correct after the fix on both dialects.
- Cover exact and late-parent arrivals and a mismatched input that belongs to
  a real transaction but spends another outpoint.
- Race enrichment with invalidation or terminal transition; a terminal spender
  must not acquire or regain spend ownership.
- If storage changes, test atomic refusal of inconsistent populated data and
  preserve valid state through reopen.

## Definition of done

Each persisted spend assignment names the exact active spending input under
the existing Wallet mutation contract, without mandating a graph redesign.

[t351]: 351-sql-utxo-integrity.md
[t410]: ../../tasks/410-transaction-graph-admission.md
