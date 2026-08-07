# Task 321 — PostgreSQL Manager and daemon-harness support

**Status:** deferred, not scheduled. Split out of [[archive/315-rebuild-minimal-dual-backend-itest-stack.md]] (archived) §4.

## Why deferred

315 supports exactly two backends at the Manager: kvdb and SQLite. That is what
the two itest commands need. Carrying PostgreSQL through the same slice was what
produced the three-live-field `DBConfig` the owner rejected.

The PostgreSQL **Store** is not deferred — it exists, is exercised by
`wallet/internal/db/itest` under the Makefile's PostgreSQL package set (build
tag `test_db_postgres`, not `pg_db_itest`), and stays green. Adding methods to
`db.Store` forces its adapter and generated code to compile, so that work stays
in #1272. What is deferred is Manager-level construction and daemon/harness
wiring.

## What to do

- add `DBBackendPostgres` as a **third switch case** inside 315's single
  `NewManager(ctx, ManagerConfig)`. Not a new constructor, not another config
  type — `DataSource` already carries a DSN and `MaxConnections` already serves
  the SQL pool. Keep the one-database-per-Manager shape: one PostgreSQL database
  shared by every wallet, distinguished by `wallet_id`;
- teach the daemon and `bwtest` harness to configure it (DSN, schema, pool
  limits), and let `make itest db=postgres` select it;
- decide whether the itest matrix runs all three backends in CI or only on
  demand — three chain backends times three databases is a large matrix.

Sequence this **after** [[322-shared-chain-reset-coordination.md]] if multiple
wallets will share one PostgreSQL database in the itests, since that is exactly
the configuration where the shared-reset gap bites.

## Related deferred work

`archive/deferred-tasks.md` holds the **native pgx cleanup**: move the
PostgreSQL backend to `pgx/v5` end-to-end by introducing backend-neutral
nullable types in `wallet/internal/db` so shared code stops carrying
`database/sql` types. That is independent of this task and can land in either
order, but doing it first avoids reworking the adapters twice.

## Definition of done

1. `make itest chain=btcd db=postgres icase=manager` passes.
2. `ManagerConfig` remains the Manager's only configuration type; PostgreSQL
   adds one `DBBackendPostgres` switch case to the existing `NewManager`, with
   no backend-specific constructor and no nested backend config.
3. The CI matrix decision is recorded, whichever way it goes.

## Evidence

`archive/deferred-tasks.md` (native pgx plan);
`archive/314-post-review-completion-of-the-three-prs.md` §4 (why config
defaulting was deleted).
