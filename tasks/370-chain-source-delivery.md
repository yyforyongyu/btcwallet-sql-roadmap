# Task 370 — fan out ordered Manager chain events

Component: **Manager chain-event fanout and Wallet processors**.

Type: **Refactor**.

Dependencies: **[Task 331][t331] and [Task 334][t334]**.

## Rationale

Every managed SQL Wallet currently consumes chain notifications independently.
Multiple Wallets can therefore compete for one notification channel instead of
receiving the same ordered events, and their workers have no shared shutdown
boundary before Manager closes the Store.

## Details

- Add one internal Manager-owned dispatcher that accepts normalized connected,
  filtered-block, and relevant-transaction events.
- Deliver each accepted event to every affected Wallet processor in source
  order. Use bounded, shutdown-aware delivery and no goroutine per event.
  Block progress reaches every registered Wallet, even with no matching
  transaction. Wallet matching must not mutate another processor's payload.
- Let Wallet processors retain wallet-scoped address, transaction, tip, and
  historical-rescan work. The dispatcher adds no wallet-specific ownership
  policy.
- Register the complete startup Wallet set and Wallets created while Manager
  is running. One failed processor follows Task 334's aggregate failure policy
  rather than silently dropping events.
- Fix registration against event admission: a new Wallet receives events
  admitted after that boundary without disturbing existing deliveries.
  Task 402 owns watch registration; Task 423 integrates it with catch-up
  at this boundary before public Create success.
- Closing admission must settle accepted delivery and join every processor
  before Task 331 closes the Store. Settle delivery as complete or failed;
  partial fanout is not success. Full queues cannot block failure/Stop or
  wait for an acknowledgment from a failed processor.
- Keep the maintained live source path unchanged in this task.
  [Task 423][t423] owns the later source cutover, [Task 371][t371] owns durable
  canonical state, and [Task 372][t372] owns database-wide rollback.
- Task 423 gates all connected and filtered-block candidates on Task 371 and
  normalizes equal block representations before this prepared fanout path.
  This task does not add a second canonical gate or deduplication mechanism.
- Exclude disconnected-event admission, a generic event bus, durable replay
  system, canonical Store mutation, rollback, targeted-rescan redesign, and
  cross-process coordination. Task 372 owns the complete disconnect path.

## Suggested fix

Use one small dispatcher with a bounded mailbox per Wallet or an equivalent
synchronous handoff. Private queue, worker, and event types are not part of the
contract.

## Verification

- Register two Wallet processors and submit mixed events directly. Both
  receive applicable events in identical order.
- Fill one delivery queue, fail that processor, and race Stop. Accepted work
  settles as delivered or failed without deadlock or leaked workers.
- Deliver a block with no matching transaction to one Wallet; its tip advances
  and processing leaves another Wallet's payload unchanged.
- Register a newly created Wallet at a documented event boundary.
- Race admission, delivery, and stop; all accepted delivery finishes before
  Store closure and later delivery is rejected.
- Prove disconnected events are not admitted through this task's prepared
  connected-event path.
- Prove the maintained source consumer remains unchanged until Task 423.

## Definition of done

One internal Manager dispatcher can fan out ordered connected-block,
filtered-block, and relevant-transaction events to the complete SQL Wallet set
and shut down safely. Task 372 remains the sole owner of disconnect admission
and rollback.

[t331]: 331-manager-wallet-lifecycle.md
[t334]: ../archive/tasks/334-shared-chain-state.md
[t371]: 371-canonical-chain-frontier.md
[t372]: 372-multi-wallet-rollback.md
[t423]: 423-manager-chain-source-cutover.md
