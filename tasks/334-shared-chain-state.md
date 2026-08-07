# Task 334 — coordinate shared chain state and rollback

Component: **Manager and Controller/Syncer**.

Dependencies: **[Task 333][t333] and [Task 340][t340], plus an accepted ADR
superseding ADR 0002**. [Task 333][t333] supplies the PostgreSQL harness; this
task owns shared-chain behavior on that harness.

## Rationale

Multiple SQL wallets share canonical blocks, but a wallet can invoke database-
wide rollback while other wallets have independent runtime state.

## Details

- Reconfirm the invariant: one canonical chain per database.
- Supersede ADR 0002 before implementation. The accepted replacement must move
  canonical chain-source ownership from each Wallet Syncer to the Manager
  coordinator while preserving the Controller/processor separation below.
- Choose one owner for canonical extension, reorg detection, and rollback.
- Give that coordinator exactly one authoritative chain source supplied through
  `ManagerConfig`. Every Wallet under the Manager uses it. Remove the duplicate
  per-Wallet chain override from the maintained construction path rather than
  trying to compare interface values or letting two sources nominate competing
  canonical tips.
- Make the coordinator the sole consumer of the chain source's notification
  channel. Normalize each connected block, disconnected block, and relevant
  transaction event once, apply its canonical database mutation once, then
  dispatch the ordered event to every affected Wallet's processor. Wallet
  syncers must not compete to receive from the shared channel.
- Keep Wallet processors responsible for wallet-scoped account/address scan
  state, credits, debits, transaction status, wallet tips, and targeted-rescan
  intent. They request verified chain data through the coordinator; they do not
  consume the authoritative source channel or mutate the canonical frontier.
  Historical scan policy remains per Wallet, while canonical block identity,
  extension, reorg, and rollback remain coordinator-owned.
- Give each per-Wallet delivery a bounded, shutdown-aware queue or synchronous
  handoff with an explicit failure policy. Manager shutdown closes the source
  consumer, stops new dispatch, and joins every admitted delivery before the
  shared Store closes; do not leak one fanout goroutine per event.
- Targeted rescans must not move the global frontier. They may request an
  idempotent coordinator-owned historical block identity needed by a confirmed
  transaction, provided height, hash, and canonical header timestamp agree.
- Canonical extension carries verified previous-block linkage. Reject a new tip
  unless its parent hash equals the current canonical tip hash; height, hash,
  and timestamp alone are insufficient.
- Define the empty-frontier bootstrap explicitly. A verified birthday block is
  historical wallet state and does not silently become a competing global
  frontier. The first coordinator-owned frontier candidate uses an atomic
  nil-frontier compare-and-set; one verified candidate wins, and a concurrent
  different candidate rereads and adopts that frontier before it may extend.
  Every later extension follows the ordinary parent-match rule.
- Preserve the current detach triggers unless a current failing reproduction
  proves them wrong; do not replace `ON DELETE SET NULL` speculatively.
- Avoid process-local write locks as the correctness boundary.
- Limit the supported runtime to one live `Manager` per database. Multiple
  writable Managers or writer processes over one database are outside this
  task's contract; do not add a lifetime database lease, cross-Manager fencing,
  or reload machinery here.
- Reconcile an ambiguous extension or rollback commit by rereading the exact
  canonical frontier and every affected wallet tip before resuming wallets.
  Return a typed indeterminate result if neither complete old nor complete new
  state can be proven.
- Do not add durable rescan jobs; a restarted targeted rescan starts over.

## Suggested fix

Prefer a Manager-level chain coordinator with one Manager-owned chain source,
a durable canonical frontier, and transaction-scoped compare-and-set. Quiesce
affected wallets for a database-wide rollback, apply detach and tip changes
atomically, then reconcile the durable result before resuming them. Route
verified linked headers and
historical block insertion through the same coordinator without advancing the
frontier. Bootstrap an absent frontier with the same coordinator CAS; never let
`UpdateWallet` independently claim it from a birthday. If operations become
wallet-scoped instead, prove other wallets remain untouched. Run one chain event
loop in the coordinator and fan out normalized ordered events through bounded
per-Wallet processors after the event's one canonical mutation.

## Verification

The superseding ADR defines the Manager coordinator and remaining per-Wallet
processor and rescan responsibilities before code lands. SQLite two-wallet
reorg tests prove one coherent result, a pre-birthday confirmation can persist
its verified block identity without moving the tip, and no wallet observes a
partially rolled-back state. Any schema detach rewrite first has a failing
Store regression test on both SQL dialects. Barrier-controlled SQLite Manager
fixtures force extension and historical insertion against rollback and produce
one serializable frontier; backend-neutral Store CAS and rollback conformance
also runs on both dialects. This task repeats the Manager fixtures on
PostgreSQL using [Task 333][t333]'s harness. Wrong-parent successors are rejected, and
ambiguous extension and rollback outcomes resolve as old, new, or typed
indeterminate before wallet execution resumes. Concurrent first-wallet tests
with equal and different verified birthdays produce exactly one frontier while
retaining each wallet's truthful historical birthday state. All Manager
concurrency fixtures use multiple wallets under one Manager; this task makes no
multi-Manager or multi-process guarantee. A construction test proves a second
Wallet cannot install a divergent source, and competing-fork fixtures prove
every Wallet follows the Manager's single authoritative source. Two Wallets
receiving the same notification both process it in order while instrumentation
proves the underlying channel had one consumer and the canonical mutation ran
once. Backpressure, wallet failure, removal, and Manager shutdown fixtures
prove the fanout cannot drop accepted events, deadlock shutdown, or leak
workers.

## Definition of done

- One Manager-owned coordinator is the sole canonical-chain source consumer
  and mutation owner for every Wallet in a database.
- Extension, bootstrap, rollback, historical insertion, fanout, and ambiguous
  outcomes preserve one serializable frontier on both SQL dialects.
- The PR makes no multi-Manager, multi-process, or durable-rescan guarantee.

[t333]: 333-postgres-manager-harness.md
[t340]: 340-durable-database-identity.md
