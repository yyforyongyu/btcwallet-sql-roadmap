# Task 371 — serialize canonical chain-frontier mutation

Component: **SQL Store canonical chain state**.

Type: **Bug fix**.

Dependencies: **[Task 334][t334] and [Task 340][t340]**.

## Rationale

The SQL Store shares block rows across Wallets but does not durably identify
the canonical frontier. Historical scan writes and live connected-block writes
can therefore disagree about which operation is allowed to advance shared
chain state.

## Details

- Persist one database-wide canonical frontier while keeping Wallet tips
  wallet-scoped.
- Atomically bootstrap an absent frontier and extend only from the expected
  height and hash with verified parent linkage. Equal candidates are
  idempotent; stale, skipped, wrong-parent, and conflicting candidates fail
  without moving it.
- Keep verified historical block insertion non-advancing.
- Return enough stable outcome information for the Manager to distinguish an
  applied or idempotent candidate from stale, conflicting, ordinary-failure,
  and indeterminate results.
- Provide an exact non-mutating frontier read for startup and recovery.
- Apply the same public Store behavior on SQLite and PostgreSQL. Reconcile an
  ambiguous commit from the exact durable frontier rather than retrying it.
- Exclude notification fanout, rollback, Wallet quiescence, generic journals
  or locks, rescan persistence, and Manager fencing.

## Suggested fix

A singleton frontier record and operation-specific compare-and-set are one
likely implementation. Another local design is acceptable if it preserves the
same durable states and classified outcomes.

## Verification

- Cover empty bootstrap, equal replay, competing first candidates, ordinary
  extension, stale expected state, wrong parent, skipped height, and
  same-height conflict on both SQL dialects.
- Race historical insertion with extension and prove history never advances
  the frontier.
- Force ambiguous bootstrap and extension outcomes and classify exact old,
  exact new, and indeterminate states without blind retry.

## Definition of done

SQLite and PostgreSQL expose one serializable canonical frontier that only
verified live extension or rollback can move, while historical writes remain
non-advancing.

[t334]: ../archive/tasks/334-shared-chain-state.md
[t340]: ../archive/tasks/340-durable-database-identity.md
