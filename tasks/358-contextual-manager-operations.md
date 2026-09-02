# Task 358 — make Manager creation cancellable

Component: **Manager.Create**.

Type: **Gap fix**.

Dependencies: **[Task 329][t329], [Task 331][t331], [Task 456][t456], and
[Task 467][t467]**.

## Rationale

`Manager.Create` uses `context.Background` for preparation and backend work.
Adding caller cancellation without one exact mutation cutoff can make a
successful durable genesis appear canceled or leave prepared secrets alive.

## Details

- Replace the public method with `Create(ctx, params)`, putting
  `context.Context` first and moving durable wallet identity into the existing
  `CreateWalletParams`. Consume Task 456's Manager-owned runtime configuration
  and migrate maintained callers without a background legacy wrapper.
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
  then uses one bounded Manager-owned context for transaction, result
  classification, assembly, and publication.
- On `runtime.ErrAmbiguousTxCommit`, fix Task 467's
  `ErrIndeterminateCommit`, enter or observe Task 331's existing terminal
  Create path, privately stop and join every unpublished candidate, and
  initiate or join Manager shutdown before returning. Never retry or publish a
  Wallet; keep exact committed-result reconciliation in Task 366.
- Destroy ordinary prepared secrets on cancellation and every definite local
  failure through this operation's existing ownership path. Task 367 owns
  exhaustive plaintext-lifetime hardening and fault probes.
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
  machine, exact-result reader, or cross-process fence.

## Suggested fix

Thread one request context to the backend-specific claim point, then switch to
one Manager-owned bounded context. Acquire Task 331's Create gate with the
request context before either backend's writer admission and release it on
every pre-claim cancellation. Give kvdb only its honest
pre-Update/callback checkpoints and documented non-cancellable wait, and map
ambiguous SQL mutation through Task 467 and Task 331's existing terminal path
without retry or publication.

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
  create classification or private-start result, and success
  returns only the fully started Wallet pointer required by Task 331.
- SQL ambiguous responses match `ErrIndeterminateCommit`, never retry or
  publish a Wallet, and leave Manager terminally stopped after private
  candidate cleanup. A queued second Create returns `ErrManagerStopped` before
  backend or cache work. Focused ownership probes prove canceled or definitely
  failed preparation destroys its ordinary secret material.
- kvdb tests document writer-wait and post-claim-scrypt limitations without a
  cancellation goroutine.
- Canceling the request after return does not stop the Wallet.

## Definition of done

- Public `Manager.Create(ctx, params)` accepts `context.Context` first, reads
  identity from `CreateWalletParams`, and honors cancellation only before the
  documented backend mutation claim, including while waiting for Task 331's
  Create gate.
- Post-claim mutation, classification, assembly, private Wallet start, and
  publication return either the completed Manager-owned result or shared
  indeterminacy after Task 331's terminal cleanup, without retry or
  publication.
- The PR contains no aggregate startup work, generic registry, legacy wrapper,
  or new reconciliation/secret contract.

[t329]: 329-atomic-wallet-account-creation.md
[t331]: 331-manager-wallet-lifecycle.md
[t456]: 456-manager-runtime-configuration.md
[t467]: ../archive/tasks/467-indeterminate-commit-identity.md
