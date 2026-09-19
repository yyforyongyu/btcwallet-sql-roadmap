# Task 349 — implement idempotent transaction invalidation

Component: **TxStore internal invalidation workflow**.

Type: **Gap fix**.

Dependencies: **[Task 339][t339], [Task 409][t409], [Task 410][t410], and
[Task 411][t411]**.

> **Superseded 2026-09-05:** SQL already has the internal wallet-scoped branch
> invalidation required by the maintained caller. Task 412 owns the remaining
> public `TxWriter` exposure without this speculative graph prerequisite.

## Rationale

Implement a narrow internal SQL operation that idempotently invalidates one
unmined root and its normalized descendants while retaining audit history.

## Details

Keep both modern kvdb and SQL public calls unsupported until Task 412 exposes
the SQL operation together with publication admission. Leave internal legacy
publisher cleanup isolated. Traverse normalized edges cycle-safely with a
documented depth/work bound and fail atomically when it is exceeded. Clear
only spend state owned by members of the invalidated result so a later valid
spender survives. Record only the minimal operation-specific result needed to
reconcile an ambiguous commit as success, definite absence, or typed
indeterminate.

Exclusions: schema/backfill, graph admission implementation, terminal-parent
ingestion, public API exposure, TxPublisher/network ordering, generic
transaction journals, replay state machines, and physical deletion.

## Suggested fix

Use a bounded recursive query over normalized edges and one
minimal invalidation-result record keyed by the operation identity; update the
member statuses and owned spend pointers atomically, then reconcile that exact
record after an ambiguous commit.

## Verification

Direct internal Store tests prove root/descendants become terminal while
siblings and later valid spenders survive; retries are idempotent; cycle/work
limits fail with no partial status; exact
committed/absent/partial/unavailable ambiguous outcomes classify on both SQL
dialects; SQL and kvdb public calls remain unsupported.

## Definition of done

SQL has one internal bounded invalidation operation over the authoritative
graph with retained history and exact ambiguous-result reconciliation; no
public exposure, publisher behavior, or generic journal is included.

[t339]: 339-transaction-visibility.md
[t409]: 409-transaction-input-graph.md
[t410]: ../../tasks/410-transaction-graph-admission.md
[t411]: ../../tasks/411-terminal-parent-rejection.md
