# Task 409 — simplify transaction input graph reads

Component: **SQL TxStore input-graph representation**.

Type: **Refactor**.

Dependencies: **[Task 340][t340] and [Task 424][t424]**.

## Rationale

Current SQL invalidation decodes raw transactions to discover descendants,
including relationships through external inputs. That already represents the
graph; normalization is not a missing public API. This deferred refactor is
justified only if Task 424 identifies a concrete consumer need that the
existing representation cannot serve adequately.

## Details

- Preserve every input relationship, wallet identity, transaction status, and
  observable invalidation, rollback, and deletion result.
- Change only the input representation or reads justified by Task 424.
  Retaining raw-transaction decoding is valid; do not require a new table,
  materialized graph, or removal of decoding merely for architectural
  uniformity.
- If the accepted solution persists new input facts, maintain them atomically
  with raw transactions on each ingestion path. Preflight existing data and
  preserve wire-range indexes, including the coinbase previous-output marker.
- Any required forward migration must refuse inconsistent source facts without
  partial cutover. Do not invent a repair that changes transaction meaning.
- Keep this optional refactor out of the hard dependency chain for Tasks 410,
  411, and 413. Their behavior must not require its preferred representation.
- If Task 424 finds no necessary change, supersede this task rather than
  implementing temporary schema work. Exclude new public APIs, concurrent
  mutation policy, spend-pointer semantics, and legacy-package changes.

## Suggested fix

Reuse the existing decoded input graph first. A narrower query or normalized
relation is an option only when the accepted evidence requires it; private
row and helper shapes are not part of this task's contract.

## Verification

- Compare existing and revised results for internal, external, and late-parent
  edges through invalidation, rollback, and leaf deletion on both dialects.
- Preserve transaction identity and exact previous outpoints across reopen.
- If storage changes, test empty/populated migrations and atomic rejection of
  inconsistent raw bytes, identities, or input facts.
- Demonstrate the concrete benefit accepted by Task 424 without adding public
  behavior or making another fix depend on the representation.

## Definition of done

The justified input-read change preserves the complete existing graph and
public results with the smallest local implementation. If no change is needed,
use supersession instead of recording implementation completion.

[t340]: ../archive/tasks/340-durable-database-identity.md
[t424]: 424-transaction-utxo-integrity-adr.md
