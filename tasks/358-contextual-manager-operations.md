# Task 358 — make Manager creation cancellable

Component: **Manager.Create**.

Dependencies: **[Task 331][t331], [Task 366][t366], and [Task 367][t367]**.

## Rationale

`Manager.Create` uses `context.Background` for preparation and backend work.
Adding caller cancellation without one exact mutation cutoff can make a
successful durable genesis appear canceled or leave prepared secrets alive.

## Details

- Replace the public method with `Create(ctx, cfg, params)`, putting
  `context.Context` first, and migrate maintained callers without a background
  compatibility wrapper.
- Carry the caller context through Task 331 admission, validation, SQL writer
  acquisition, and safe preparation checkpoints.
- Define successful claim of the backend create transaction as the mutation
  cutoff. Cancellation before the claim wins and leaves no durable or cached
  state; at or after the claim, the caller context no longer controls the
  result.
- SQL acquires writer admission with the caller context, arbitrates once,
  marks the transaction claimed, and then uses one bounded Manager-owned
  context for transaction, reconciliation, assembly, and publication.
- Consume Task 366's committed, absent, or indeterminate genesis result and
  Task 367's prepared-secret destruction. Do not redefine either contract.
- kvdb checks cancellation before `walletdb.Update` and on callback entry
  before atomically claiming mutation. Document that its writer-lock wait and
  post-claim scrypt are not cooperatively cancellable; do not race them with a
  cancellation goroutine.
- After the cutoff, finish assembly and Task 331 admission-protected
  publication and return the definitive result. A returned Wallet has a
  lifetime independent of the request context.
- Exclude `Manager.Load`; Task 422 owns its reversible work and
  publication cutoff.
- Do not add a generic operation registry, create manifest, publication state
  machine, or cross-process fence. Task 329 remains transitive through Tasks
  366 and 367.

## Suggested fix

Thread one request context to the backend-specific claim point, then switch to
one Manager-owned bounded context and reuse Tasks 366 and 367. Give kvdb only
its honest pre-Update/callback checkpoints and documented non-cancellable
wait.

## Verification

- Barriers cover admission, SQL writer acquisition, every preparation
  checkpoint, and both backend claim points.
- Pre-claim cancellation leaves no durable row, kvdb namespace, cached
  Wallet, or prepared secret. Post-claim cancellation cannot replace the
  definitive create/reconciliation result.
- SQL ambiguous committed/absent/indeterminate outcomes match Task 366, and
  Task 367 probes prove all canceled prepared secrets are destroyed.
- kvdb tests document writer-wait and post-claim-scrypt limitations without a
  cancellation goroutine.
- Canceling the request after return does not stop the Wallet.

## Definition of done

- Public Manager creation accepts `context.Context` first and honors it only
  before the documented backend mutation claim.
- Post-claim mutation, reconciliation, assembly, and publication return one
  definitive Manager-owned result.
- The PR contains no Load cancellation, generic registry, compatibility
  wrapper, or new reconciliation/secret contract.

[t331]: ../archive/tasks/331-manager-wallet-lifecycle.md
[t366]: 366-reconcile-sql-genesis-commit.md
[t367]: 367-bound-sql-genesis-secrets.md
