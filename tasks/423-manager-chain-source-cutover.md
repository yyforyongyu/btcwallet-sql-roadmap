# Task 423 — cut over Manager chain notification ownership

Component: **SQL Manager chain source and dispatcher wiring**.

Type: **Refactor**.

Dependencies: **[Task 370][t370], [Task 372][t372], [Task 402][t402],
and [Task 456][t456]**.

## Rationale

After fanout and rollback are ready, SQL Wallets still register and consume
chain notifications independently. Manager becomes the sole Wallet
notification consumer so every active Wallet receives the same ordered
stream. Shutdown joins Manager's reader and delivery before Store closure.

## Details

- Reuse the existing ManagerConfig.ChainSource instance. This replaces
  Task 456's earlier source-factory requirement. Manager owns the Wallet
  notification reader and registration, not the client's connection. Remove
  competing per-Wallet readers; the caller keeps the source usable through
  Manager Stop and closes it afterward. Align ADR 0015 with this ownership,
  Wallet checkpoints, and background catch-up.
- Make Manager the sole caller of live notification registration and
  `Notifications()`, forwarding normalized events through Tasks 370 and 372.
- Recover from each Wallet's existing processed height/hash, committed
  atomically with its block effects. Use the chain backend for catch-up, not
  a separate persisted Manager frontier, admission gate, or reconciliation.
  Ambiguous processing writes follow the terminal Manager failure policy.
- Treat connected and filtered notifications for the same height and hash as
  representations of one block. Coalesce their block application without
  dropping relevant transaction payloads or applying a Wallet effect twice.
  Reuse existing event normalization; add no durable deduplication framework.
- Before Start/Create publishes an operational Wallet, establish Task 402's
  watches and Task 370's no-gap delivery boundary. Retain or backpressure live
  events across this handoff; fail if it cannot be maintained. Block catch-up
  cannot recover missed mempool events.
- Manager Running does not mean every Wallet is synced. Permit Start/Create
  to return with watch-ready Wallets catching up in the background; each
  Wallet reports its own truthful synced state. Preserve Task 372's stale
  historical-write protection without forcing all Wallet tips to align.
- Drive Task 402's watch reconstruction and registration through the same
  Wallet processing boundary, preserving its Manager-wide failure policy
  without adding a durable delivery mechanism.
- On Stop, close reader admission, settle accepted delivery, and join the
  reader and dispatcher before Store closure, even if the borrowed client's
  channel stays open. Reuse Task 331's drain and cancellation rules.
- Keep modern kvdb on its isolated single-Wallet source path. Preserve
  Wallet-scoped targeted rescans without letting them consume the live stream
  or move another Wallet's checkpoint.
- One active Manager per database remains caller responsibility. Optional
  duplicate-Manager enforcement in Task 457 does not gate this cutover.
- Exclude a generic event bus, durable rescan jobs, individual Wallet
  lifecycle, daemon changes, and cross-process leadership.

## Suggested fix

Move the current SQL notification loop to Manager and connect it to the
prepared dispatcher. Startup buffering and Wallet registration mechanics are
private choices as long as the public no-gap and shutdown boundaries hold.

## Verification

- Start empty, one-Wallet, and two-Wallet Managers with current, behind, and
  off-chain tips on SQLite and PostgreSQL. Return only after watch/delivery
  readiness, while a behind Wallet may remain unsynced and catch up without
  blocking a current Wallet.
- Interleave connected, filtered-block, transaction, and disconnected events
  with startup, Create, registration, and Stop. Every normalized effect is
  delivered once in order and no Wallet misses a cutover boundary.
- Exercise connected-only, filtered-only, and equal mixed representations.
  No block effect is duplicated and no filtered transaction is lost.
- Prove only Manager consumes the SQL stream, targeted rescans remain scoped,
  modern kvdb remains isolated, and Manager's reading and delivery work ends
  before Store closure.
- Interrupt delivery after only one Wallet applies a block; restart recovers
  each Wallet from its own checkpoint without skipped or duplicated effects.
- Exercise a payment arriving during startup/new-Wallet handoff and before
  mining; background catch-up must not hide a lost mempool event.
- Stop with an empty open source channel and with full delivery. The reader
  joins without stopping the borrowed client or waiting for channel closure.

## Definition of done

One running SQL Manager owns the Wallet notification reader and ordered
delivery to its complete Wallet set without competing per-Wallet readers or
stopping the borrowed chain client.

[t370]: 370-chain-source-delivery.md
[t372]: 372-multi-wallet-rollback.md
[t402]: 402-replay-live-watch-obligations.md
[t456]: ../archive/tasks/456-manager-runtime-configuration.md
