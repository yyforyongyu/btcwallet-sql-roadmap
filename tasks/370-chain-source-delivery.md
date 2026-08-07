# Task 370 — prepare ordered Manager chain delivery

Component: **Manager chain coordinator and Wallet processors**.

Dependencies: **[Task 331][t331], [Task 333][t333], [Task 334][t334], and
[Task 371][t371]**.

## Rationale

Every managed Wallet currently retains its own chain source, registers block
notifications, and consumes the source notification channel. A Manager-owned
coordinator needs bounded ordered Wallet delivery and canonical forward-event
admission before it can replace that maintained path. Activating the source
takeover before the complete rollback path exists would make reorg handling
unsafe between implementation PRs.

## Details

- Add an internal Manager chain coordinator with a registry of bounded,
  shutdown-aware per-Wallet processors. Give the coordinator explicit start,
  stop, and join ownership without registering for or consuming live source
  notifications in this task.
- Define the internal normalized event and result shapes needed for connected,
  filtered-block, and relevant-transaction delivery. Reserve disconnected
  admission for Task 372 rather than installing an incomplete fallback.
- Treat every block-bearing connected or filtered-block event as one canonical
  candidate. Invoke Task 371's completed canonical extension contract exactly
  once for that candidate before any block-bearing fanout, coalescing equal
  connected and filtered representations rather than advancing twice.
  Dispatch only after an applied extension or idempotent exact-candidate
  result proves the durable state.
- A stale frontier, mismatch, ordinary failure, or typed indeterminate result
  admits no Wallet delivery and halts the prepared coordinator. Resume only
  after an explicit durable-frontier reread establishes a determinate next
  candidate; never retry the cached candidate from an ambiguous response.
- Deliver every accepted forward event to each affected Wallet processor in
  order through a bounded queue or the exact synchronous handoff accepted by
  Task 334. Use at most one long-lived delivery worker per Wallet, never one
  goroutine per event.
- Let Wallet processors retain ownership of address/account scan state,
  transaction ownership resolution, wallet tips, and targeted scans.
- Preserve Store-time ownership resolution for relevant transactions so one
  event can be offered safely to multiple Wallet processors without the
  coordinator inventing wallet-specific transaction policy.
- Provide the stop-and-join contract that [Task 423][t423] can compose with
  Task 331's Manager close owner. Do not wire it into maintained Manager
  admission or Store closure yet.
- Handle direct Wallet registration, removal, backpressure, and processor
  failure according to the accepted ADR. One failed processor cannot silently
  drop an accepted event or deadlock coordinator shutdown.
- Use Task 333's PostgreSQL Manager harness only for backend-neutral
  conformance. Add no PostgreSQL-specific coordinator behavior.
- Leave maintained Manager and Wallet source selection, notification
  registration, and `Notifications()` consumption unchanged until Task 423
  atomically activates the coordinator after Task 372 is complete.
- Exclude live source takeover, disconnected-event admission, rollback,
  canonical frontier schema, durable watch replay, durable rescan jobs,
  generic event buses, and cross-Manager fencing.

## Suggested fix

Build one internal coordinator that accepts directly submitted normalized
forward events and dispatches them through bounded per-Wallet processors. Gate
every block-bearing connected or filtered-block fanout on Task 371's final
classified Store result and expose an explicit lifecycle plus the narrow
extension point Task 372 needs. Keep all maintained notification wiring on the
existing path until Task 423.

## Verification

- Instantiate the prepared coordinator directly with two Wallet processors
  and submit mixed connected, filtered, and transaction events. Both receive
  their applicable events in identical order.
- Inject stale, mismatch, ordinary-failure, and indeterminate canonical
  outcomes for connected and filtered-block candidates. Neither the candidate
  nor later block-bearing events reach a Wallet until an explicit
  fresh-frontier recovery establishes continuation.
- Submit a filtered block without a paired connected event, then submit equal
  connected and filtered representations. The first advances the frontier
  before fanout and the equal pair performs exactly one canonical mutation.
- Block one processor at queue capacity and exercise the ADR's exact
  backpressure/failure policy without dropping accepted events or spawning
  event-scoped goroutines.
- Stop or remove one Wallet while another continues and prove direct delivery
  neither targets the removed processor nor disrupts the surviving order.
- Race direct coordinator stop with event admission and Wallet delivery. Stop
  joins every accepted delivery with no Store access afterward.
- Run focused coordinator tests and repeat the two-Wallet direct-delivery case
  on SQLite and PostgreSQL through Task 333's harness.
- Add a regression assertion that maintained Manager/Wallet construction still
  uses the existing source registration and notification-consumption path.

## Definition of done

- One internal Manager coordinator supplies bounded ordered forward events to
  all directly registered Wallet processors.
- Block-bearing connected and filtered-block events reach Wallet processors
  only after Task 371 proves their canonical extension; this task owns
  arbitration and fanout but no frontier mutation implementation.
- The coordinator has explicit lifecycle, recovery, backpressure, and Wallet
  removal behavior without worker leaks or lost accepted events.
- Maintained source ownership and reorg behavior remain unchanged; Task 423
  performs the later atomic cutover only after Task 372 completes rollback.
- The PR changes no rollback mutation, live source registration, watch replay,
  targeted-rescan durability, or PostgreSQL-specific Wallet behavior.

[t331]: 331-manager-wallet-lifecycle.md
[t333]: 333-postgres-manager-harness.md
[t334]: 334-shared-chain-state.md
[t371]: 371-canonical-chain-frontier.md
[t423]: 423-manager-chain-source-cutover.md
