# Task 457 — enforce one process-local Manager per database

Component: **Manager Start and process-local database ownership**.

Type: **Gap fix**.

Dependencies: **[Task 368][t368]**.

## Rationale

The aggregate lifecycle assumes that one Manager owns the Store and chain work
for a database. Two Manager instances in one process can otherwise both Start,
publish Wallets, consume chain work, and close shared durable state without a
common owner. The supported contract needs a narrow process-local guarantee;
cross-process leadership remains a separate problem.

## Details

- Define a non-secret canonical database-target key from validated
  `ManagerConfig`. Canonicalize filesystem paths for kvdb and SQLite. Parse and
  normalize PostgreSQL server, database, and schema identity while excluding
  passwords and pool or tuning options from registry state, logs, and errors.
- Acquire one process-local lease during the winning Manager Start before any
  Wallet, chain worker, or Running state is published. A competing Manager for
  the same canonical target returns one stable wallet-owned
  `ErrManagerAlreadyRunning` and starts no work.
- Treat lease rejection as a terminal Start failure under Task 331. Close the
  losing Manager's unpublished runtime and Store before returning the error; it
  cannot later retry Start on the same instance.
- Key ownership by the Manager instance, not only by a boolean. Repeated Start
  on the winning Manager follows Task 331's idempotent result and cannot reject
  itself or acquire a second lease.
- Release the lease only after failed-start unwind or Manager Stop has joined
  every Wallet and shared worker and closed the Store. A replacement Manager
  cannot Start while old teardown can still access the database.
- Permit Managers for different canonical database targets to run concurrently
  in one process, including different temporary databases on the same network.
- Keep the registry synchronized and release test owners only through the same
  owner-checked teardown path. Expose no public registry or mutable global
  configuration.
- State the exact guarantee: syntactically equivalent supported configuration
  forms normalize to one key, but alternate host aliases or external routing
  that cannot be proven equivalent remain caller responsibility.
- Exclude database rows, TTLs, heartbeats, PostgreSQL advisory locks, file-lock
  protocols, multi-process detection, distributed leadership, and fencing
  tokens.

## Suggested fix

Add a package-private synchronized registry from canonical target key to one
opaque Manager owner token. Acquire through one compare-and-insert called by
Task 331's Start owner and release with an owner-checked delete at the final
teardown boundary. Keep the canonical key and raw DataSource out of public
errors and logging.

## Verification

- Race Start on two Managers for the same kvdb, SQLite, and PostgreSQL target.
  Exactly one reaches Running; the loser returns
  `ErrManagerAlreadyRunning` and publishes no Wallet or goroutine.
- Prove supported equivalent path and PostgreSQL DSN forms collide without
  retaining or logging credentials.
- Start Managers for distinct targets concurrently and prove neither blocks
  the other.
- Pause old Manager teardown before worker join, Store close, and lease release.
  A replacement cannot Start until every old resource is gone.
- Inject startup failure before and after partial Wallet activation. The lease
  remains held through unwind and is released exactly once afterward.
- Race repeated Start and Stop on the owning Manager and prove no stale or
  foreign owner can release its lease.

## Definition of done

- At most one Manager per canonical configured database target can be Running
  or starting in one process.
- Lease acquisition precedes all runtime publication and owner-checked release
  follows complete teardown and Store closure.
- Same-owner repeated calls and distinct databases behave independently.
- Errors and registry state expose no database credentials.
- The PR makes no cross-process, distributed-leadership, advisory-lock, or
  filesystem-fencing claim.

[t368]: 368-wallet-startup-shutdown.md
