# Task 457 — enforce one process-local Manager per database

Component: **Manager Start and process-local database ownership**.

Type: **Feature**.

Dependencies: **[Task 331][t331]**.

## Rationale

The aggregate lifecycle assumes that one Manager owns the Store and chain work
for a database. Two Manager instances in one process can otherwise both Start,
publish Wallets, consume chain work, and close shared durable state without a
common owner. Callers are currently responsible for running one Manager per
database. This optional Hardening task adds misuse protection; it does not
gate fan-out or lnd integration. Cross-process leadership remains separate.

## Details

- Identify the same configured database across supported equivalent path and
  PostgreSQL DSN forms. Passwords and pool options must not distinguish owners
  or appear in diagnostics. Document unsupported external aliases; do not build
  a general database-identity resolver.
- Reject a competing SQL Manager during Start, before publishing Wallets or
  chain workers, with wallet-owned `ErrManagerAlreadyRunning`. Keep kvdb's
  existing exclusive-open protection; do not add another ownership layer or
  change its legacy open errors.
- Treat ownership rejection as a terminal Start failure under Task 331. Close
  the losing Manager's unpublished runtime and Store before returning the error;
  it cannot later retry Start on the same instance.
- Repeated Start/Stop on the same Manager preserves its lifecycle contract;
  it cannot reject itself as a competing owner or release another owner.
- Allow replacement only after failed-start unwind or Manager Stop has joined
  every Wallet and shared worker and closed the Store. A replacement Manager
  cannot Start while the old terminal stop can still access the database.
- Permit Managers for different canonical database targets to run concurrently
  in one process, including different temporary databases on the same network.
- Keep bookkeeping private and race-safe, including test cleanup. Add no
  public registry or mutable global configuration.
- Exclude database rows, TTLs, heartbeats, PostgreSQL advisory locks, file-lock
  protocols, multi-process detection, distributed leadership, and fencing
  tokens.

## Suggested fix

Reuse existing database identity and lifecycle boundaries. A small process-
local ownership guard is sufficient where backend exclusion is absent; its
private representation and synchronization are implementation choices.

## Verification

- Race Start on two Managers for the same SQLite and PostgreSQL target.
  Exactly one reaches Running; the loser returns
  `ErrManagerAlreadyRunning` and publishes no Wallet or goroutine.
- For kvdb, verify its existing exclusive-open protection prevents a second
  live owner without requiring two successfully constructed Managers.
- Prove supported equivalent path and PostgreSQL DSN forms collide without
  retaining or logging credentials.
- Start Managers for distinct targets concurrently and prove neither blocks
  the other.
- Pause old Manager Stop before worker join and Store close.
  A replacement cannot Start until every old resource is gone.
- Inject startup failure before and after partial private Wallet start.
  Replacement remains excluded until unwind finishes.
- Race repeated Start and Stop on the owning Manager and prove no stale or
  foreign owner can clear another Manager's ownership.

## Definition of done

- At most one Manager per canonical configured database target can be Running
  or starting in one process.
- Ownership exclusion precedes runtime publication and lasts through complete
  Manager Stop and Store closure, reusing kvdb's exclusive-open behavior.
- Same-owner repeated calls and distinct databases behave independently.
- Errors and ownership bookkeeping expose no database credentials.
- The PR makes no cross-process, distributed-leadership, advisory-lock, or
  filesystem-fencing claim.

[t331]: ../archive/tasks/331-manager-wallet-lifecycle.md
