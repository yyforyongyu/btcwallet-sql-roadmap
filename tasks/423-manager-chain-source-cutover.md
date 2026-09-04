# Task 423 — cut over Manager chain notification ownership

Component: **SQL Manager chain source and dispatcher wiring**.

Type: **Refactor**.

Dependencies: **[Task 370][t370], [Task 371][t371], [Task 372][t372],
[Task 402][t402], [Task 456][t456], and [Task 457][t457]**.

## Rationale

After fanout and rollback are ready, SQL Wallets still register and consume
chain notifications independently. The Manager must become the single live
source owner in one cutover so every active Wallet receives the same ordered
stream and shutdown can join it before Store closure.

## Details

- Construct one authoritative SQL chain source from Task 456's Manager-owned
  configuration. Remove per-Wallet source construction from the maintained
  Manager path.
- Make Manager the sole caller of live notification registration and
  `Notifications()`, forwarding normalized events through Tasks 370 and 372.
- Gate every connected or filtered-block candidate on Task 371's canonical
  frontier before Task 370 fanout. Deliver only after an applied or idempotent
  result; keep delivery quiesced on stale, conflicting, failed, or
  indeterminate results.
- Treat connected and filtered notifications for the same height and hash as
  representations of one block. Coalesce their block application without
  dropping relevant transaction payloads or applying a Wallet effect twice.
  Reuse existing event normalization; add no durable deduplication framework.
- During Manager Start, register the complete Wallet set, align the canonical
  frontier and Wallet tips with the source, then publish Running. Buffer or
  reject live events until that no-gap boundary is established.
- Register a newly created Wallet at a closed delivery boundary before its
  public Create result is returned.
- Drive Task 402's existing live-watch replay through the same Wallet
  processing boundary; do not add another replay mechanism.
- On Stop, close source admission, settle accepted delivery, and join the
  source and dispatcher before Store closure.
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
  with startup, Create, replay, and Stop. Every normalized effect is delivered
  once in order and no Wallet misses a cutover boundary.
- Exercise connected-only, filtered-only, and equal mixed representations.
  Every block advances or confirms the frontier before Wallet delivery, with
  no duplicate block effect or lost filtered transaction. No processor sees a
  block whose frontier result is stale, conflicting, failed, or indeterminate.
- Prove only Manager consumes the SQL stream, targeted rescans remain scoped,
  modern kvdb remains isolated, and no source work outlives Store ownership.

## Definition of done

One running SQL Manager owns the live chain source and ordered delivery to its
complete Wallet set, with no per-Wallet notification consumer on the maintained
path.

[t370]: 370-chain-source-delivery.md
[t371]: 371-canonical-chain-frontier.md
[t372]: 372-multi-wallet-rollback.md
[t402]: 402-replay-live-watch-obligations.md
[t456]: ../archive/tasks/456-manager-runtime-configuration.md
[t457]: 457-process-local-manager-lease.md
