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
  Task 402 owns watch readiness; Task 423 joins this boundary with live and
  historical delivery. Watch-ready Wallets may catch up in the background.
- Closing admission must settle accepted delivery and join every processor
  before Task 331 closes the Store. Settle delivery as complete or failed;
  partial fanout is not success. Full queues cannot block failure/Stop or
  wait for an acknowledgment from a failed processor.
- Keep the maintained live source path unchanged in this task.
  [Task 423][t423] owns the later source cutover and equal-block notification
  normalization; [Task 372][t372] owns database-wide rollback.
- Keep each Wallet's processed height/hash atomic with its own block effects.
  Recovery uses those existing checkpoints and the chain backend, not a
  separately persisted Manager frontier.
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

[t331]: ../archive/tasks/331-manager-wallet-lifecycle.md
[t334]: ../archive/tasks/334-shared-chain-state.md
[t372]: 372-multi-wallet-rollback.md
[t423]: 423-manager-chain-source-cutover.md
