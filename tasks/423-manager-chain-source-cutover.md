# Task 423 — cut over Manager chain notification ownership

Component: **SQL Manager chain source and dispatcher wiring**.

Type: **Refactor**.

Dependencies: **[Task 370][t370], [Task 371][t371], [Task 372][t372],
[Task 402][t402], [Task 456][t456], and [Task 457][t457]**.

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
  Manager Stop and closes it afterward. Align ADR 0015 with this ownership.
- Make Manager the sole caller of live notification registration and
  `Notifications()`, forwarding normalized events through Tasks 370 and 372.
- Gate every connected or filtered-block candidate on Task 371's canonical
  frontier before Task 370 fanout. Deliver only after an applied or idempotent
  result; keep delivery quiesced on stale, conflicting, failed, or
  indeterminate results until catch-up proves safe progress or Manager stops.
  Recover lagging Wallet tips after a frontier commit and interrupted delivery
  using existing catch-up, not an event journal.
- Treat connected and filtered notifications for the same height and hash as
  representations of one block. Coalesce their block application without
  dropping relevant transaction payloads or applying a Wallet effect twice.
  Reuse existing event normalization; add no durable deduplication framework.
- During Manager Start, register the complete Wallet set, align the canonical
  frontier and Wallet tips with the source, then publish Running. Retain or
  backpressure live events until that boundary is established; fail startup
  if it cannot be maintained. Block catch-up cannot recover missed mempool
  events.
- Catch up a newly created Wallet and register its processor at Task 370's
  delivery boundary before public Create success.
- Drive Task 402's watch reconstruction and registration through the same
  Wallet processing boundary, preserving its Manager-wide failure policy
  without adding a durable delivery mechanism.
- On Stop, close reader admission, settle accepted delivery, and join the
  reader and dispatcher before Store closure, even if the borrowed client's
  channel stays open. Reuse Task 331's drain and cancellation rules.
- Keep modern kvdb on its isolated single-Wallet source path. Preserve
  Wallet-scoped targeted rescans without letting them consume the live stream
  or move the canonical frontier.
- Exclude a generic event bus, durable rescan jobs, individual Wallet
  lifecycle, daemon changes, and cross-process leadership.

## Suggested fix

Move the current SQL notification loop to Manager and connect it to the
prepared dispatcher. Startup buffering and Wallet registration mechanics are
private choices as long as the public no-gap and shutdown boundaries hold.

## Verification

- Start empty, one-Wallet, and two-Wallet Managers with current, behind, and
  off-chain tips on SQLite and PostgreSQL. Publish Running only after exact
  alignment or fail without partial activation.
- Interleave connected, filtered-block, transaction, and disconnected events
  with startup, Create, registration, and Stop. Every normalized effect is
  delivered once in order and no Wallet misses a cutover boundary.
- Exercise connected-only, filtered-only, and equal mixed representations.
  Every block advances or confirms the frontier before Wallet delivery, with
  no duplicate block effect or lost filtered transaction. No processor sees a
  block whose frontier result is stale, conflicting, failed, or indeterminate.
- Prove only Manager consumes the SQL stream, targeted rescans remain scoped,
  modern kvdb remains isolated, and Manager's reading and delivery work ends
  before Store closure.
- Crash after frontier commit but before one Wallet applies the block;
  restart catches up its missing effects before resuming live delivery.
- Stop with an empty open source channel and with full delivery. The reader
  joins without stopping the borrowed client or waiting for channel closure.

## Definition of done

One running SQL Manager owns the Wallet notification reader and ordered
delivery to its complete Wallet set without competing per-Wallet readers or
stopping the borrowed chain client.

[t370]: 370-chain-source-delivery.md
[t371]: 371-canonical-chain-frontier.md
[t372]: 372-multi-wallet-rollback.md
[t402]: 402-replay-live-watch-obligations.md
[t456]: ../archive/tasks/456-manager-runtime-configuration.md
[t457]: 457-process-local-manager-lease.md
