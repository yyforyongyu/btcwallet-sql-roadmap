# Task 422 — make Manager loading cancellable

Component: **Manager.Load**.

Type: **Gap fix**.

Dependencies: **[Task 331][t331]**.

## Rationale

`Manager.Load` performs Store opening and Wallet assembly with
`context.Background`. Unlike Create, it has no durable mutation cutoff; its
only irreversible library event is publication of the assembled Wallet.

## Details

- Replace the public method with `Load(ctx, cfg)`, putting
  `context.Context` first, and migrate maintained callers without a background
  compatibility wrapper.
- Carry the caller context through Task 331 admission, backend loading, and
  Wallet assembly checkpoints.
- Use Task 331's per-name admission and cache publication as the one
  linearization owner. A cancellation observed before publication closes all
  resources opened by that attempt and publishes nothing.
- Arbitrate cancellation immediately before publication while holding the
  per-name admission. Once publication wins, return that Wallet regardless of
  later caller cancellation; never discard or overwrite an existing winner.
- A cache hit returns the already-published Wallet after normal admission and
  closed-state checks; it does not rebuild or take ownership of that Wallet.
- Derive the Wallet lifetime independently from the request context.
- Keep Manager close, Store ownership, same-name serialization, and losing
  assembly cleanup in Task 331.
- Exclude Create's mutation cutoff, SQL genesis reconciliation, secret
  destruction, generic operation registries, and publication state machines.

## Suggested fix

Thread the request context through backend load and assembly, then use one
Task 331 per-name critical section to choose cancellation or publication.
Close attempt-owned resources on every pre-publication cancellation/error and
return the published pointer on every post-publication path.

## Verification

- Cancellation barriers cover Manager admission, backend Store load,
  assembly, and immediately before and after cache publication.
- Every pre-publication cancellation closes attempt-owned resources and leaves
  the cache unchanged; every post-publication cancellation returns the
  published Wallet.
- Same-name Load races produce one Wallet pointer and no losing resource leak.
- Close-versus-Load ordering remains Task 331's contract, and canceling the
  Load context after return does not stop the Wallet.

## Definition of done

- Public Manager loading accepts `context.Context` first and remains
  cancellable until its one admission-protected publication point.
- No canceled losing assembly leaks a Store or Wallet, and no winning
  publication is discarded because its request context later ends.
- The PR contains no Create/genesis behavior, compatibility wrapper, generic
  registry, or publication state machine.

[t331]: 331-manager-wallet-lifecycle.md
