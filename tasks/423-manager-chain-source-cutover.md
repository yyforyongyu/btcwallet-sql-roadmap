# Task 423 — cut over SQL Manager chain notification ownership

Component: **SQL Manager chain source adapter and coordinator cutover**.

Type: **Refactor**.

Dependencies: **[Task 370][t370], [Task 372][t372], [Task 402][t402],
[Task 412][t412], [Task 456][t456], and [Task 457][t457]**.

## Rationale

SQL Wallets still register and consume chain sources separately after Tasks 370
and 372 prepare complete forward and rollback processing. The SQL Manager
cannot become the sole live notification owner before the aggregate lifecycle,
process-local ownership, transaction publication ordering, and both prepared
chain paths are complete. A partial cutover could lose ordered events or leave
one Wallet outside database-wide rollback.

## Details

- Construct exactly one authoritative chain source from Task 456's immutable
  Manager-owned factory. Remove the pre-cutover per-Wallet factory calls from
  maintained managed construction. Do not add a compatibility field or modify
  CLI, RPC, deprecated-wallet, lnd, or daemon production code.
- Make the SQL Manager source adapter the sole maintained consumer of
  `Notifications()` and the sole owner of live block-notification
  registration. Keep modern kvdb on its documented isolated source path.
- Start the source and coordinator inside aggregate Manager Start after Task
  457 acquires database ownership and before Task 331 publishes Running.
  Register every Wallet returned by startup assembly as one complete active
  set; there is no Load, dormant Wallet, or individual Start path.
- Drive [Task 402][t402]'s owner-neutral delivery/replay component through the
  same coordinator. Register the startup Wallet set before Running, attach a
  new Wallet before Create publication, and close admission and settle accepted
  replay during Stop. Do not add a second replay component or Store state.
- Aggregate the earliest durable history requirement across every Wallet and
  supply that optimization hint before source startup. Never substitute the
  canonical frontier for Wallet history requirements.
- Begin in a buffering state. Reconcile Task 371's exact canonical frontier,
  every durable Wallet tip, and the source's exact current best chain before
  releasing buffered notifications or publishing Running.
- If the frontier is absent, prove all durable Wallet tips share one persisted
  ancestry and bootstrap at the highest verified tip. With no durable tip, use
  the source's exact current best. Refuse unverifiable or divergent ancestry.
- If the durable frontier is off the source chain, use Task 372 once to move
  the frontier and every affected Wallet to a verified common ancestor. Then
  submit each parent-linked forward block through Task 370 until the frontier
  reaches the source best.
- Register a Wallet created while Running under closed coordinator admission.
  Align its durable tip to the current proven frontier, deliver the complete
  closed-admission suffix, and atomically publish its processor with Task 331's
  fully started Create result before reopening admission.
- Do not unregister or stop one Wallet while Manager remains Running. A
  processor failure follows Task 334's aggregate failure policy and cannot
  silently remove that Wallet or let later accepted events pass it.
- Preserve targeted rescans through one restricted coordinator request for
  verified historical chain data. Such results remain Wallet-scoped and never
  enter the notification stream or move the canonical frontier.
- On Manager Stop, close source admission, finish or reject every accepted
  event, and stop new delivery before Task 331 privately stops Wallets and
  joins their processors. Join the source/coordinator after accepted delivery
  settles, then allow Task 331 to close Store. No source event or targeted
  response may outlive Store ownership.
- Exclude durable rescan jobs, a generic event bus, idle-Wallet eviction,
  individual lifecycle, process-external leadership, and cross-process
  fencing.

## Suggested fix

Move the maintained SQL notification registration into one Manager-owned
adapter. During Manager Start, buffer source events, reconcile the durable
frontier and all Wallet tips, catch up through the prepared rollback and
forward paths, register the complete active set, and publish Running only after
the no-gap boundary is established. Reuse the same closed-admission boundary
for Create-time processor registration and aggregate Stop.

## Verification

- Start empty, one-Wallet, and two-Wallet SQLite and PostgreSQL Managers with
  absent, current, behind, and off-chain durable frontiers. Startup publishes
  either one fully aligned active set or no Running Manager.
- Interleave connected, filtered, transaction, and disconnected notifications
  with startup buffering. Every accepted event applies once in source order and
  no Wallet misses the catch-up/live boundary.
- Create a Wallet while the source advances. Its public Create result appears
  only after exact alignment and atomic processor registration; existing
  Wallets retain ordered delivery.
- Force forward failure, rollback failure, ambiguous commit, divergent Wallet
  ancestry, processor failure, and source shutdown. Admission remains closed
  unless exact durable state proves safe continuation.
- Race Manager Stop with source receipt, Create-time registration, rollback,
  targeted history, and Wallet delivery. Every accepted operation joins before
  Vault lock and Store closure.
- Pre-seed pending Task 402 obligations at startup, Create registration, and
  Stop. The same Wallet component replays or settles each obligation without a
  gap, duplicate driver, or accepted delivery surviving shutdown.
- Prove only the Manager consumes the SQL notification stream, targeted rescans
  cannot move the frontier, and the modern-kvdb source path remains isolated.

## Definition of done

- One running SQL Manager owns the only live source consumer and the complete
  active Wallet processor set.
- Startup and Create establish explicit no-gap alignment before publishing
  Wallet availability; there is no Load or individual registration path.
- Forward delivery, reorg rollback, targeted history, processor failure, and
  aggregate shutdown follow the accepted ADR without lost or duplicated work.
- Manager Stop joins all source and Wallet work before Store closure.
- The PR adds no individual lifecycle, idle eviction, generic replay system,
  daemon change, or cross-process guarantee.

[t331]: 331-manager-wallet-lifecycle.md
[t402]: 402-replay-live-watch-obligations.md
[t370]: 370-chain-source-delivery.md
[t372]: 372-multi-wallet-rollback.md
[t412]: 412-transaction-publication-admission.md
[t457]: 457-process-local-manager-lease.md
[t456]: 456-manager-runtime-configuration.md
