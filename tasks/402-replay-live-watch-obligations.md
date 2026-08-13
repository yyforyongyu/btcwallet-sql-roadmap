# Task 402 — replay live-watch obligations through the Syncer

Component: **Controller/Syncer watch initialization and delivery completion**.

Type: **Gap fix**.

Dependencies: **[Task 364][t364] and [Task 401][t401]**.

## Rationale

Persisted obligations are not useful until one runtime owner
seeds a coherent watch snapshot, replays later generations in order, and
completes pending results after live-tip delivery.

## Details

- Before live sync, load one coherent persisted watch-chain snapshot and its
  generation, register it through the live-tip primitive, then replay later
  obligations in generation/order.
- Keep chain I/O outside Store transactions and mark completion idempotently
  only after chain success.
- Resume the same pending delivery after restart, chain failure, or
  cancellation without dropping or reordering accepted obligations.
- Provide the smallest wallet-owned pending/delivery handle needed by
  AddressManager producers to await or retry completion without exposing an
  undelivered script.
- Exclude producer mutations, historical rescans, key-only locators, and a
  general notification framework.

## Suggested fix

Add one Syncer-owned replay loop driven by retained Task 364
queries and Task 401, plus operation-specific waiters keyed by stable
delivery ID.

## Verification

Startup barriers, ordered multi-generation replay, duplicate
delivery, chain failure, cancellation, and restart prove no accepted obligation
is lost and no waiter completes before live registration.

## Definition of done

The Syncer is the sole runtime owner of ordered,
restart-safe SQL live-watch delivery and pending completion, with no historical
scan or producer mutation in the PR.

[t364]: 364-durable-watch-registration.md
[t401]: 401-live-tip-chain-watch.md
