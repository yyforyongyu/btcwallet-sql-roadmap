# Task 412 — expose invalidation with publication ordering

Component: **TxWriter public contract, TxPublisher, and transaction graph
admission**.

Dependencies: **[Task 349][t349]**.

## Rationale

Expose SQL transaction invalidation only when network publication is ordered
against its durable decision.

## Details

Use [Task 410][t410]'s graph admission, reached transitively through Task 349,
with Task 349's internal durable invalidation status/result so invalidation
can atomically reject a pre-send transition. Add the SQL TxWriter public
operation in this same PR, after both sides use that admission boundary. Keep
the modern kvdb public call unsupported. Never hold a database transaction
across the network call. Preserve current already-known/confirmed behavior and
exact publisher cleanup errors.

Persist one publication-specific attempt marker with the exact wallet,
transaction identity, and bytes before releasing admission for the network
call. A repeated call or restart must settle that marker before any new send or
invalidation: network-known means finalize the same attempt as sent;
definitively absent means retry the same bytes under a renewed admission
decision; indeterminate network state retains the marker and returns a typed
settlement-indeterminate result. Invalidation cannot overtake an unsettled
attempt, and retry cannot substitute different bytes for its identity.

Exclusions: invalidation traversal or result semantics, graph schema, generic
write locks, generic replay workers or state machines, network/RPC redesign,
and publisher performance work.

## Suggested fix

Add a durable pre-send transition under Task 410 graph admission, release the
database transaction before network I/O, and make Task 349 invalidation
atomically reject or order against that transition. Expose the SQL TxWriter
call only after this complete path is wired. Reconcile an existing attempt by
checking the exact transaction's chain/mempool visibility with a bounded
context, then finalize, retry the same send, or preserve the fence as
indeterminate according to the operation-specific rules above.

## Verification

Before this PR, SQL and kvdb public invalidation calls remain unsupported. In
this PR, the SQL public call becomes available only with a
publication-versus-invalidation barrier that permits the two documented
serial orders; no order sends a transaction after durable invalidation;
network errors preserve the original error plus exact cleanup state; kvdb
remains unsupported. Crash injection immediately before send, after send, and
before durable finalization proves restart settlement never loses a successful
send, duplicates different bytes, or lets invalidation cross an unresolved
attempt. Unknown settlement keeps the durable fence and returns its typed
result.

## Definition of done

The SQL public invalidation operation and publication admission become
available atomically, publication cannot cross invalidation between durable
admission and send, every crash boundary has deterministic retry or
indeterminate settlement, and no database transaction is held during network
I/O.

[t349]: 349-transaction-invalidation.md
[t410]: 410-transaction-graph-admission.md
