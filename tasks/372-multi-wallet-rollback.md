# Task 372 — prepare atomic multi-Wallet rollback

Component: **Manager chain coordinator and SQL Store rollback**.

Type: **Bug fix**.

Dependencies: **[Task 349][t349], [Task 370][t370], and [Task 371][t371]**.

## Rationale

`Store.RollbackToBlock` is already database-wide and atomic, but any one
Wallet worker can invoke it while other Wallet processors continue reading or
writing the affected chain and transaction graph. The prepared Manager
coordinator needs one complete disconnect path before [Task 423][t423] can
safely make it the authoritative live notification owner.

## Details

- Extend Task 370's internal coordinator with disconnected-event admission and
  make it the only owner of rollback within that prepared path. Direct Wallet
  processors receive no raw disconnect and do not call the database-wide
  mutation through the new path.
- Stop coordinator admission for the rollback boundary and quiesce every
  affected Wallet processor in stable wallet-ID order. Join accepted
  chain and transaction mutations before opening the rollback transaction.
- Consume [Task 410][t410]'s transaction-graph admission, reached transitively
  through Task 349, in the same stable wallet-ID order. Task 349 owns only the
  invalidation semantics and result used by rollback. Do not add a generic
  Store write lock or acquire only the reporting Wallet's gate.
- Preserve the existing one-transaction Store owner for shared block deletion,
  transaction detachment, coinbase/descendant status changes, and affected
  wallet-tip rewinds. Narrow or extend it only where the accepted frontier
  contract requires exact old/new state and result reporting.
- Roll back from a verified fork point and update the canonical frontier and
  every affected durable Wallet tip atomically with the detach. While Manager
  is Running, every durable Wallet has a processor and participates in runtime
  quiescence.
- Preserve the current foreign-key detach strategy unless a focused failing
  Store reproduction proves a schema change is required. Do not speculatively
  replace `ON DELETE SET NULL` or unrelated `RESTRICT` constraints.
- After every Store result, determine whether the accepted disconnect was
  durably applied. A proven new state may continue, but a definite pre-commit
  failure, an ambiguous result resolved to the complete old state, or an
  unresolved ambiguous result all leave the consumed disconnect unapplied.
- For every outcome that does not prove the disconnect durably applied, keep
  affected processors and coordinator admission quiesced, reacquire a fresh
  verified source-ancestry snapshot, and redrive from that snapshot. Do not
  reuse a stale fork point or resume merely because the old state is coherent.
- On an ambiguous commit, reread the exact canonical frontier, block range,
  every affected Wallet tip, and the operation-specific transaction graph
  result. Classify only complete old or new states; return typed indeterminate
  and remain stopped when neither is proven.
- Resume prepared coordinator admission and affected processors only after
  exact reconciliation proves the disconnect durably applied. Publish one
  post-rollback frontier result and never fan out the raw disconnect as a
  second rollback trigger.
- Keep manual `RewindWallet` and targeted rescans wallet-scoped. They never
  delete shared blocks, invoke database-wide rollback, move the canonical
  frontier, or change another Wallet's live readiness.
- Leave the maintained source consumer and its existing reorg behavior
  unchanged. Task 423 cuts over to this completed rollback path together with
  Task 370's forward path in one later cutover PR.
- Exclude live source takeover, forward frontier implementation, durable rescan
  jobs, generic replay markers, multi-Manager leases, and cross-process
  fencing.

## Suggested fix

Add one directly invokable rollback method to Task 370's prepared coordinator.
It pauses admission and delivery, acquires Task 410's graph admission in stable
order, and calls a narrowly extended `RollbackToBlock` result contract. Keep
the Store mutation in the existing backend-neutral workflow and do not connect
the coordinator to the live notification channel until Task 423.

## Verification

- Directly invoke the prepared coordinator with two SQLite Wallets while one
  processor write is blocked. Rollback waits for both processors, mutates no
  partial state, updates both tips and the canonical frontier atomically, then
  resumes in order.
- Verify one submitted disconnect causes exactly one coordinator-owned Store
  rollback, processors receive no raw disconnect, and one post-rollback
  frontier result is published.
- Repeat backend-neutral Store rollback conformance on PostgreSQL through Task
  333's completed harness path; no dialect-specific Wallet behavior is allowed.
- Race direct rollback with coordinator delivery, canonical extension,
  historical insertion, transaction insertion, invalidation, Create-time
  Wallet registration, and coordinator stop. Stable admission ordering must
  avoid deadlock.
- Force definite pre-commit failure and ambiguous commit outcomes. Complete
  new state resumes once; definite failure, complete old state, and mixed or
  unresolved state keep admission closed and require a fresh verified source
  snapshot before redrive. Mixed state returns typed indeterminate and resumes
  no affected processor.
- Verify manual `RewindWallet` changes only its target Wallet and never moves
  the canonical frontier or deletes shared blocks.
- Confirm current detach triggers on both SQL dialects. Add a migration only if
  the focused reproduction first demonstrates a real constraint defect.
- Assert maintained notification ownership and reorg processing remain on the
  pre-cutover path after this task merges.

## Definition of done

- Task 370's prepared coordinator owns one directly tested database-wide
  rollback path after all affected processors and transaction-graph writers
  are quiescent in stable order.
- Shared block removal, transaction detachment, canonical frontier movement,
  and every affected durable Wallet tip form one atomic mutation on both SQL
  dialects.
- Every outcome that does not prove the disconnect durably applied retains
  quiescence and redrives only from a fresh verified source snapshot; an
  unverifiable result is typed indeterminate and remains safely quiesced.
- Maintained notification and reorg behavior remain unchanged until Task 423
  atomically cuts over to the complete forward and rollback paths.
- The PR adds no live source cutover, generic Store lock, replay framework,
  speculative detach rewrite, durable rescan job, or cross-process guarantee.

[t349]: 349-transaction-invalidation.md
[t370]: 370-chain-source-delivery.md
[t371]: 371-canonical-chain-frontier.md
[t410]: 410-transaction-graph-admission.md
[t423]: 423-manager-chain-source-cutover.md
