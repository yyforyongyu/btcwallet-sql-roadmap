# Task 358 — make Manager creation cancellable

Component: **Manager.Create**.

Type: **Gap fix**.

Dependencies: **[Task 331][t331], [Task 366][t366], and [Task 367][t367]**.

## Rationale

`Manager.Create` uses `context.Background` for preparation and backend work.
Adding caller cancellation without one exact mutation cutoff can make a
successful durable genesis appear canceled or leave prepared secrets alive.

## Details

- Replace the public method with `Create(ctx, identity, params)`, putting
  `context.Context` first and consuming Task 456's Manager-owned runtime
  configuration. Migrate maintained callers without a background legacy
  wrapper.
- Carry the caller context through validation, context-aware entry to Task
  331's Manager/Create admission and distinct Create gate, SQL writer
  acquisition, and safe preparation checkpoints.
- Acquire the Create gate before SQL writer admission, kvdb update admission,
  or any durable mutation. Cancellation while waiting for the gate wins and
  leaves no durable or cached state. After acquisition, cross Task 331's sole
  Create acceptance boundary by atomically rechecking `Running` under the
  Manager state gate; waiting for admission does not accept the Create.
- Define successful claim of the backend create transaction as the mutation
  cutoff. Cancellation before the claim wins and leaves no durable or cached
  state; at or after the claim, the caller context no longer controls the
  result. This caller-cancellation cutoff is independent of Task 331's
  Stop-versus-Create acceptance boundary: after lifecycle acceptance, Stop
  cannot replace the Create's operation result with `ErrManagerStopped`.
- After context-aware Create-gate acquisition, SQL acquires writer admission
  with the caller context, arbitrates once, marks the transaction claimed, and
  then uses one bounded Manager-owned context for transaction, reconciliation,
  assembly, and publication.
- Consume Task 366's committed, absent, or indeterminate genesis result and
  Task 367's prepared-secret destruction. Do not redefine either contract.
- kvdb checks cancellation before `walletdb.Update` and on callback entry
  before atomically claiming mutation. Document that its writer-lock wait and
  post-claim scrypt are not cooperatively cancellable; do not race them with a
  cancellation goroutine.
- After the cutoff, finish assembly, Task 331's required synchronous private
  Wallet start, and Manager-owned active-set publication before returning the
  definitive result. A returned Wallet has a lifetime independent of the
  request context and retains no Manager context.
- Exclude aggregate Manager startup discovery and cancellation; Task 459 owns
  attempt-local assembly and Task 331 owns startup publication.
- Do not add a generic operation registry, create manifest, publication state
  machine, or cross-process fence. Task 329 remains transitive through Tasks
  366 and 367.

## Suggested fix

Thread one request context to the backend-specific claim point, then switch to
one Manager-owned bounded context and reuse Tasks 366 and 367. Acquire Task
331's Create gate with the request context before either backend's writer
admission and release it on every pre-claim cancellation. Give kvdb only its
honest pre-Update/callback checkpoints and documented non-cancellable wait.

## Verification

- Barriers cover admission, Create-gate acquisition, SQL writer acquisition,
  every preparation checkpoint, and both backend claim points.
- On kvdb, SQLite, and PostgreSQL, hold one Create in private Wallet start and
  cancel a second caller while it waits for Task 331's Manager/Create gate.
  The second call has not crossed the Create acceptance boundary; it returns
  its context error and reaches no backend writer admission, durable mutation,
  or cache mutation.
- Pre-claim cancellation leaves no durable row, kvdb namespace, cached
  Wallet, or prepared secret. Post-claim cancellation cannot replace the
  definitive create, private-start, or reconciliation result, and success
  returns only the fully started Wallet pointer required by Task 331.
- SQL ambiguous committed/absent/indeterminate outcomes match Task 366, and
  Task 367 probes prove all canceled prepared secrets are destroyed.
- kvdb tests document writer-wait and post-claim-scrypt limitations without a
  cancellation goroutine.
- Canceling the request after return does not stop the Wallet.

## Definition of done

- Public Manager creation accepts `context.Context` first and honors it only
  before the documented backend mutation claim, including while waiting for
  Task 331's Create gate.
- Post-claim mutation, reconciliation, assembly, private Wallet start, and
  publication return one definitive Manager-owned result.
- The PR contains no aggregate startup work, generic registry, legacy wrapper,
  or new reconciliation/secret contract.

[t331]: 331-manager-wallet-lifecycle.md
[t366]: 366-reconcile-sql-genesis-commit.md
[t367]: 367-bound-sql-genesis-secrets.md
[t456]: 456-manager-runtime-configuration.md
