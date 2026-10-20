# Task 422 — make Manager loading cancellable

Component: **Manager.Load**.

Type: **Gap fix**.

Dependencies: **[Task 331][t331], [Task 368][t368], and
[Task 453][t453]**.

This task was superseded before implementation on 2026-08-28, when the roadmap
adopted Task 368's planned removal of public Manager loading. Task 459 owns the
distinct aggregate-startup assembly work. The original task identity and scope
remain here as historical evidence.

## Rationale

`Manager.Load` performs Store opening and Wallet assembly with
`context.Background`. Unlike Create, it has no durable mutation cutoff; its
only irreversible library event is publication of the assembled Wallet.

## Details

- Replace the public method with `Load(ctx, cfg)`, putting
  `context.Context` first, and migrate maintained callers without a background
  legacy wrapper.
- Carry the caller context through Task 331 close admission, Task 453's
  per-name entry gate, backend loading, and Wallet assembly checkpoints.
- Use Task 453's cold-load publication point and Task 368's terminal-replacement
  publication point as the exact linearization owners. A cancellation observed
  before the applicable publication closes all resources opened by that
  attempt and publishes nothing.
- Arbitrate cancellation immediately before publication while holding the
  per-name admission. Once publication wins, return that Wallet regardless of
  later caller cancellation; never discard or overwrite an existing winner.
- A cache hit returns the exact current Wallet only while its runtime is
  `Created`, `Starting`, or `Started`, after normal admission and closed-state
  checks. A `Stopping` entry joins terminal completion under the Load context
  and then follows Task 368's fresh-pointer replacement path; a `Stopped` entry
  enters that replacement path immediately. Load never returns a terminal
  runtime pointer.
- Derive the Wallet lifetime independently from the request context.
- Keep Manager close admission and Store ownership in Task 331. Keep cold
  same-name serialization, Create-versus-Load publication ordering, and losing
  assembly cleanup in Task 453. Keep terminal replacement in Task 368.
- Exclude Create's mutation cutoff, SQL genesis reconciliation, secret
  destruction, generic operation registries, and publication state machines.

## Suggested fix

Thread the request context through backend load and assembly, then use Task
453's per-name publication gate, as extended by Task 368 for terminal
replacement, to choose cancellation or publication under Task 331's close
admission. Branch on the exact entry state under that same gate: return a
created or active pointer, join a stopping runtime, or replace a terminal
runtime.
Close attempt-owned resources on every pre-publication cancellation/error and
return the published pointer on every post-publication path.

## Verification

- Cancellation barriers cover Manager admission, backend Store load,
  assembly, and immediately before and after cache publication.
- Every pre-publication cancellation closes attempt-owned resources and leaves
  the cache unchanged; every post-publication cancellation returns the
  published Wallet.
- Same-name Load races produce one Wallet pointer and no losing resource leak.
- Cache hits in `Created`, `Starting`, and `Started` return the exact current
  pointer. Hits in `Stopping` join completion before replacement, hits in
  `Stopped` replace immediately, and neither replacement path can return the old
  pointer. Cancellation while joining publishes no replacement.
- Close-versus-Load ordering remains Task 331's contract, and canceling the
  Load context after return does not stop the Wallet.

## Definition of done

- Public Manager loading accepts `context.Context` first and remains
  cancellable until its one admission-protected publication point.
- Load returns only a created or active current pointer. A stopping entry joins
  terminal completion, and stopping and stopped entries both use the
  fresh-pointer replacement path.
- No canceled losing assembly leaks a Store or Wallet, and no winning
  publication is discarded because its request context later ends.
- The PR contains no Create/genesis behavior, legacy wrapper, generic
  registry, or publication state machine.

[t331]: 331-manager-wallet-lifecycle.md
[t368]: 368-wallet-startup-shutdown.md
[t453]: 453-serialize-manager-wallet-publication.md
