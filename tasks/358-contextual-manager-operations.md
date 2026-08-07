# Task 358 — make Manager creation and loading cancellable

Component: **Manager**.

Dependencies: **[Task 329][t329] and [Task 331][t331]**.

## Rationale

Public `Manager.Create` and `Manager.Load` perform Store and backend work with
`context.Background`, so callers cannot cancel admission, safe preparation
checkpoints, or loading. Adding a context without defining the mutation cutoff
would instead make successful durable creation look canceled.

## Details

- Replace the public methods with `Create(ctx, cfg, params)` and
  `Load(ctx, cfg)`, with `context.Context` first. Update their maintained
  callers together; do not keep background-context compatibility wrappers
  without a concrete external consumer.
- Thread the caller context through [Task 331][t331] admission, validation, safe
  preparation checkpoints, and Load. SQL performs Vault preparation before its
  transaction claim; context-free production scrypt remains joined, and
  cancellation destroys its result at the next checkpoint before SQL mutation.
  Do not detach cryptographic work in a goroutine.
- Derive Wallet lifetime independently. Cancellation after a successful return
  must not stop or poison the published Wallet.
- Define one exact mutation cutoff: successful claim of the backend create
  transaction. Before that claim, caller cancellation wins and leaves no
  durable or cached state. At and after the claim, caller cancellation no
  longer controls the attempt.
- For SQL, acquire the connection or serialized writer admission with the caller
  context, arbitrate cancellation once, atomically mark the transaction claimed,
  then begin, write, commit, reconcile, assemble, and publish with one bounded
  Manager-owned context. Do not pass the caller context to `BeginTx`, because
  its later cancellation can roll back or poison `Commit` after the cutoff.
- For kvdb, check cancellation immediately before `walletdb.Update` and again
  on callback entry before atomically marking the transaction claimed. Its
  writer-lock wait is not cooperatively cancellable, so make that limitation
  explicit. Canonical `waddrmgr.Create` performs scrypt after this claim inside
  the transaction; do not split that retiring kvdb operation for cancellation.
  After the claim, complete the mutation and reopen or reconcile with the
  Manager-owned context. Do not race the callback with a cancellation goroutine.
- Make admission-protected cache publication the `Load` linearization point.
  Check cancellation immediately before publication while holding the per-name
  admission. Cancellation observed before that point closes resources opened by
  the losing attempt and publishes nothing. Once publication wins, return the
  published Wallet regardless of later caller cancellation; never overwrite or
  discard a same-name winner.
- Keep Manager close/admission ordering in [Task 331][t331]. Do not add a generic
  operation registry or publication state machine.

## Potential fix

Change the two public signatures after [Task 329][t329] and [Task 331][t331] land, then carry one
request context through admission and safe preparation checkpoints. Give SQL an
operation-specific caller-cancellable writer claim followed by a Manager-
context transaction. Give kvdb an explicit pre-`Update` checkpoint and document
its non-cancellable writer wait and post-claim scrypt. Both backends use the same
bounded Manager context for mutation, reconciliation, runtime assembly, and
publication after the cutoff. For Load, arbitrate cancellation under [Task 331][t331]'s
per-name admission immediately before the one cache-publication point.

## Done when

Canceled admission, caller-cancellable SQL writer acquisition, every safe
preparation checkpoint, ambiguous commit, initial-account genesis, and same-name
load races have focused tests. Cancellation before the transaction claim leaves
no durable or cached state; cancellation after it cannot interrupt mutation,
reconciliation, assembly, or publication. A barrier at each backend's exact
cutoff proves the arbitration. A cancellation arriving during production scrypt
in SQL is observed at the next checkpoint, its output is destroyed, and no
mutation starts. A kvdb barrier proves cancellation before the callback claim
leaves no artifact, while cancellation during post-claim scrypt cannot replace
the definitive create result. The kvdb writer-wait limitation is documented and
tested without a cancellation goroutine. Exact committed genesis returns
success despite transport cancellation; unprovable state is typed indeterminate.
Load barriers immediately before and after cache publication prove the former
returns cancellation with no retained assembly and the latter returns the
published Wallet. A returned Wallet remains usable after its request context is
canceled.

[t329]: 329-atomic-wallet-account-creation.md
[t331]: 331-manager-wallet-lifecycle.md
