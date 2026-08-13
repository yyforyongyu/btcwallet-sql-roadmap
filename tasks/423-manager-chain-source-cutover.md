# Task 423 — cut over SQL Manager chain notification ownership

Component: **SQL Manager chain source adapter and coordinator activation**.

Type: **Refactor**.

Dependencies: **[Task 368][t368], [Task 370][t370], [Task 372][t372],
[Task 412][t412], and [Task 422][t422]**.

## Rationale

SQL-managed Wallets still select, register, and consume chain sources
separately after Tasks 370 and 372 prepare complete forward and rollback
processing. The SQL Manager cannot become the sole live notification owner
before both paths, publication ordering, cancellable loading, and generation-
safe Wallet lifecycle exist, because any partial cutover could lose ordered
events, register a stale Wallet, or disable reorg handling between PRs.

## Details

- Add exactly one authoritative chain source to `ManagerConfig` and retain it
  on the Manager. The maintained Manager Create/Load path overwrites or rejects
  per-Wallet source selection; keep `Config.Chain` only for deprecated or
  non-Manager constructors that still require compatibility.
- Make the Manager source adapter the sole maintained caller consuming
  `Notifications()` and the sole owner of block-notification registration.
- Preserve the source's non-notification duties at the ownership cutover.
  Expose one restricted coordinator path for a Wallet processor to request
  verified historical chain data for a targeted scan. Bind each response to
  one verified source-ancestry snapshot, return it only to the requesting
  Wallet path, and never admit it as a notification or move Task 371's
  canonical frontier.
- Own source optimization hints at the Manager. Before source startup,
  aggregate the earliest durable history requirement across every managed
  Wallet, including stopped Wallets, and supply that minimum to the source.
  Never substitute the canonical frontier or the set of currently running
  processors for the aggregate.
- Start that sole consumer in a buffering state. Before admitting a buffered
  or newly received event, read Task 371's exact canonical frontier, every
  durable Wallet tip, and the source's exact current best chain. Do not let an
  arbitrary first connected notification bootstrap an absent frontier.
- When the frontier is absent, first prove that all durable Wallet tips lie on
  one persisted ancestry and bootstrap Task 371 at its highest verified tip;
  use the source's exact current best only when no Wallet has a durable tip.
  If that bootstrapped tip is absent from the current source chain, find a
  verified common ancestor, invoke Task 372 once to move the now-present
  frontier and all affected durable tips to that ancestor, and continue with
  forward catch-up. Refuse divergent durable ancestries or any sequence whose
  old tip, fork point, or source ancestry cannot be verified.
- When a valid durable frontier is behind the source best, submit every exact
  parent-linked block in the intervening source range through Task 370 before
  releasing buffered notifications. Each candidate advances Task 371 once and
  fans out to the currently registered processors, so blocks produced before
  registration cannot leave the next live event with a height gap.
- If the durable frontier is ahead of one or more registered Wallet tips,
  replay the verified canonical range to those Wallet processors without
  advancing the frontier again. This alignment-only catch-up must use the same
  ordered processor path and must complete before live delivery begins,
  covering a prior frontier-before-fanout crash.
- If the source chain no longer contains the durable frontier, find a verified
  common ancestor, run exactly one Task 372 rollback, and submit the forward
  catch-up range through Task 370 before releasing buffered notifications.
  Deduplicate buffered events against the reconciled frontier and preserve
  their source order.
- Keep admission closed and notifications buffered, or halt with a typed
  failure when the frontier, Wallet tips, source ancestry, rollback, or
  catch-up cannot be reconciled exactly. Never expose a partially aligned set
  of Wallet processors.
- Keep Manager Load inactive. It may construct and publish a stopped Wallet in
  the Manager cache, but it must not align the Wallet, start a processor, or
  register any generation with live chain delivery.
- Make each Wallet Start generation close coordinator admission before
  alignment. Classify the stored Manager frontier and that Wallet's durable
  tip against one verified source-ancestry snapshot. Reconcile an offline
  source reorg to a verified common ancestor first, and roll the starting
  Wallet back when its own tip is off the reconciled ancestry.
- Align the pending Wallet to the reconciled frontier without advancing that
  frontier. Keep admission closed while advancing the shared frontier to the
  exact source best, and make the unregistered pending generation an explicit
  recipient of the complete prepared forward suffix; do not rely on Task 370's
  registered-set fanout. Verify its durable tip equals the final unchanged
  frontier before atomically registering the exact generation and reopening
  admission. On failure, cancel and join it without registration.
- Make Wallet Stop close admission, unregister the exact Start generation,
  cancel and join its processor, and complete only after that generation can
  receive no later event. A concurrent or repeated Stop joins Task 368's
  authoritative generation completion.
- Normalize connected, disconnected, filtered-block, and
  relevant-transaction notifications once and submit them in source order to
  the completed coordinator.
- Route connected, filtered-block, and relevant-transaction events through
  Task 370's completed admission and Wallet-delivery path. Every block-bearing
  filtered notification is the same canonical candidate as its connected
  representation and must pass Task 371 classification before fanout. Route
  every disconnect through Task 372's completed quiescence and rollback path.
- Do not admit publication across rollback invalidation. Task 412 must complete
  before this live activation so public transaction results observe the same
  graph admission and ordering as disconnect processing.
- Activate forward and disconnect routing in the same PR. Remove the old
  maintained per-Wallet source consumers only after the complete coordinator
  path is wired; leave no configuration in which the source is claimed while
  disconnects are rejected or permanently halted.
- Integrate with Task 331's Manager close owner: close source admission, cancel
  the source consumer, and join the coordinator before Store closure.
- Preserve the accepted ADR's Wallet load, start, stop, removal, backpressure,
  and processor-failure policies while the live source is active.
- Preserve deprecated and non-Manager constructor behavior outside the
  maintained Manager path. Do not voluntarily extend the cutover to unrelated
  daemon or RPC surfaces.
- Exclude new frontier or rollback mutation semantics, Wallet processing
  policy, durable watch replay, durable rescan jobs, generic event buses,
  multi-Manager leases, and cross-process fencing.

## Suggested fix

Add one Manager-owned source loop that registers once and buffers normalized
notifications while it reconciles the exact source chain, canonical frontier,
and Wallet tips. Bootstrap an absent frontier from one verified persisted
ancestry before any rollback, use the completed rollback path for an offline
fork, advance a valid frontier through every missing source block, and replay
already-canonical ranges to registered Wallets that lag. Keep Load inactive;
make Start reconcile and catch its pending generation up under closed
admission before atomic registration, and make Stop unregister and join that
exact generation. Route targeted historical requests through a verified,
frontier-neutral coordinator method and initialize source hints from the
earliest durable requirement across all managed Wallets. Release the
deduplicated buffer only after exact active-processor alignment, switch
maintained SQL Wallet construction and shutdown to that adapter in the same
change, then remove the competing maintained per-Wallet consumers.

## Verification

- Construct two Wallets with attempted divergent per-Wallet sources and prove
  the Manager source is the only accepted maintained source.
- Give running and stopped Wallets different history requirements and prove
  source startup receives their aggregate earliest requirement, not the
  canonical frontier or earliest running-Wallet value.
- Request a targeted historical range while live delivery continues. The
  coordinator verifies it against one source snapshot and returns it only to
  the requesting Wallet; the notification consumer count, other Wallets, and
  canonical frontier remain unchanged. A mismatched ancestry is refused.
- Deliver mixed connected, disconnected, filtered-block, and transaction
  events. Both Wallet processors observe their applicable post-admission
  results in source order while `Notifications()` has one consumer and source
  registration runs once.
- Verify connected delivery uses Task 370's classified frontier result and a
  disconnect performs exactly one Task 372 rollback before ordered processing
  resumes.
- Start with an absent frontier and divergent Wallet heights on one source
  ancestry. Alignment chooses the highest verified durable baseline, catches
  lower Wallets up, and never lets the first buffered connect choose state.
- Reproduce a prior frontier-before-fanout crash with one Wallet tip behind the
  canonical frontier. Startup replays the missing verified range exactly once
  before live events reach any Wallet.
- Start with a valid frontier behind the source best and no buffered event for
  the intervening blocks. Startup advances and fans out the complete verified
  range before the next live connect, preserving Task 371's next-height rule.
- Complete a source reorg while the Manager is offline, then race new
  notifications with startup. The adapter buffers them, rolls back once to the
  verified common ancestor, applies forward catch-up, and releases only the
  deduplicated ordered suffix.
- Start with an absent frontier whose persisted Wallet ancestry is on an
  offline fork. Startup bootstraps the verified old tip, rolls back once to the
  common ancestor, catches up to the exact source best, and releases no event
  before the sequence completes.
- Leave one Wallet stopped while the frontier advances, then Load it and prove
  Load only publishes a stopped cached Wallet with no processor registration.
  Start it while the source frontier changes: admission remains closed, the
  pending generation receives the exact alignment and forward suffix before
  atomic registration, and it cannot observe a live event from a stale tip.
- Combine both conditions: start a stopped Wallet whose tip lags a stored
  frontier that is no longer on the source chain. Reconcile the offline reorg
  and rollback first, align the pending generation to the reconciled frontier,
  deliver the complete closed-admission suffix while advancing to source best,
  prove its durable tip equals the final frontier, and only then register it.
- Race Stop with source receipt and Start registration. Stop unregisters,
  cancels, and joins the exact generation before completion; a repeated Stop
  joins that result and no stopped generation receives another event.
- Deliver a filtered block with no paired connected event, then equal paired
  representations. The first advances the canonical frontier before fanout
  and the pair produces one frontier mutation while its filtered payload is
  delivered once in source order.
- Exercise a fork during steady-state delivery; neither startup nor
  steady-state can expose a forward-only, partially aligned, or permanently
  halted coordinator.
- Stop or remove one Wallet while another continues and prove source ownership
  and surviving delivery order remain intact.
- Race Manager Close with source receipt, rollback, and Wallet delivery. Task
  331's Store-close barrier observes the source loop and every accepted
  coordinator operation joined first.
- Run focused mock-chain Manager tests and repeat the two-Wallet Manager case
  on SQLite and PostgreSQL through Task 333's completed harness.
- Verify deprecated and non-Manager constructors retain their documented
  compatibility source behavior.

## Definition of done

- One Manager-owned source registration and consumer supplies all maintained
  Wallet processors through the completed coordinator.
- Forward and disconnected notifications have complete admitted paths at the
  instant maintained source ownership changes; no merge interval disables
  reorg processing.
- Startup reconciles the source chain, canonical frontier, and every loaded
  Wallet processor before live delivery, including an absent frontier, a
  valid frontier behind source best, a Wallet left behind by
  frontier-before-fanout failure, and a reorg completed while offline.
- Manager Load remains inactive. A stopped Wallet cannot join live delivery
  until Start aligns and catches its pending generation up against one verified
  source snapshot and the final unchanged canonical frontier.
- Per-Wallet maintained source selection and competing consumers are removed
  without changing deprecated or non-Manager compatibility paths.
- Targeted scans retain a coordinator-mediated verified historical-query path,
  and source optimization uses the earliest durable history requirement across
  all managed Wallets without changing the canonical frontier.
- Manager shutdown joins source consumption and every accepted coordinator
  operation before Store closure.
- The PR adds no new frontier, rollback, watch-replay, rescan, generic-bus, or
  cross-process behavior.

[t368]: 368-wallet-startup-shutdown.md
[t370]: 370-chain-source-delivery.md
[t372]: 372-multi-wallet-rollback.md
[t412]: 412-transaction-publication-admission.md
[t422]: 422-manager-load-cancellation.md
