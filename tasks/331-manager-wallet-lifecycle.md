# Task 331 — fence Manager admission and Store closure

Component: **Manager**.

Type: **Bug fix**.

Dependencies: **[Task 368][t368]**.

## Rationale

After Task 368, Manager owns each terminal Wallet runtime and all public Start
and Stop entry points, but Manager Close still needs one boundary around
backend admission and runtime teardown. Without that boundary, Create, Load,
or StartWallet can publish work while Close stops a stale snapshot or closes
the shared backend. The legacy one-wallet kvdb path can also race Create and
Load across different requested names.

## Details

- Define one public wallet-owned `ErrManagerClosed` sentinel for Create, Load,
  and StartWallet after Manager closure. Close is terminal for the Manager;
  no later operation can load or start another runtime over its backend.
- Give Manager one admission owner. Before backend work, Create and Load
  either join the admitted-operation count or return `ErrManagerClosed`;
  admission cannot be added after Close marks Manager closed.
- Reuse Task 453's per-name publication entry and Task 368's exact-pointer
  lifecycle extension. Do not add a second startup lease, lifecycle state
  machine, Wallet callback, or parallel cache owner.
- Serialize the entire legacy one-wallet kvdb Create and Load admission path
  across requested names. Keep durable duplicate-create authority and
  `ErrWalletExists` in Task 330; this task supplies only the in-memory
  admission fast path that Task 330 consumes.
- Admit StartWallet through the same Manager close boundary until its exact
  lifecycle request is accepted or rejected by the Task 368 entry. Release
  admission before waiting for the Start result. If StartWallet wins submission
  admission, Close must stop and join that exact runtime whether it is created,
  starting, or started. If Close wins, StartWallet returns `ErrManagerClosed`
  without publishing setup or workers.
- Make one Close owner atomically close admission, wait for every previously
  admitted Create and Load to publish or fail and every admitted StartWallet to
  submit or reject, stop every exact current runtime through its Manager-owned
  coordinator, join every terminal completion, and only then close the shared
  backend. Concurrent or repeated Close calls join the same completion and
  return its result.
- Permit StopWallet calls already targeting current entries to join teardown
  while Close is in progress; they cannot reopen admission or replace a
  runtime. Manager Close remains the final owner of backend closure regardless
  of which caller first requested a Wallet stop.
- Never wait for admitted operations, Wallet setup, worker teardown, or Close
  completion while holding the admission gate or Manager cache lock. Close
  takes its stable runtime snapshot only after admission is closed, Create and
  Load work has settled, and admitted StartWallet submissions have drained.
- Preserve Manager ownership of Store and legacy backend resources. Task 368
  terminalizes runtime Wallet pointers but does not close their shared Store;
  Close releases the backend only after every runtime coordinator has exited.
- Exclude Task 358's context parameters and cancellation cutoff, Task 330's
  durable duplicate classification, Task 338's accepted passphrase operation,
  repeated Unlock behavior, arbitrary non-lifecycle Wallet call fencing,
  generic operation registries, create manifests, and cross-process Manager
  fencing.

## Suggested fix

Add a small Manager close gate containing the closed bit, admitted-operation
completion, and one Close completion and result. Use the gate to serialize
admission against Close, then release it before backend work or waits. Retain
Task 453's per-name publication entry with Task 368's lifecycle extension as
the sole runtime cache owner.

Close first rejects new admission, joins admitted Create and Load work, and
drains admitted StartWallet submissions. It then submits an internal terminal
Stop to every exact current runtime entry and joins their completions before
closing the backend. StartWallet releases admission once its request is
accepted or rejected and waits for its result outside the gate, so
Close-versus-Start ordering needs no Manager-owned lease inside Wallet.

## Verification

- Add a kvdb cross-name Create and Load admission race proving the deprecated
  backend never admits two live wallet assemblies.
- Race Close against Create and Load on every maintained backend. Work admitted
  before the close point must settle before backend closure; later work must
  return `ErrManagerClosed`; no runtime may publish after backend closure.
- Race Close against StartWallet before and after lifecycle request admission.
  An admitted Start either publishes one complete runtime that Close then
  stops or is canceled and joined during startup; a rejected Start returns
  `ErrManagerClosed` without setup or worker publication.
- Pause an admitted StartWallet before coordinator submission and prove Close
  cannot take its runtime snapshot or close the backend until that submission
  accepts or rejects. Prove Close does not wait for the Start caller's response
  before submitting its terminal Stop.
- Race external StopWallet with Close and prove both join the same Task 368
  terminal completion without duplicate Vault lock, worker wait, or backend
  closure.
- Use an ordered fake backend and Wallet completion barriers to prove Close
  invokes backend close only after admitted work settles and every exact
  runtime coordinator exits. Concurrent Close calls must return the same
  result and close the backend exactly once.
- Run the focused Manager races under
  `make unit-race pkg=wallet case=TestManager...`, followed by
  `make unit pkg=wallet`.

## Definition of done

- Manager rejects later Create, Load, and StartWallet admission and joins all
  earlier admitted work before backend closure.
- Close terminalizes and joins every exact current Wallet runtime through Task
  368, then closes the Manager-owned backend last and exactly once.
- No startup lease or Wallet-local lifecycle owner is needed to order Start
  against Close, and no lifecycle race publishes against a closed Store.
- The PR adds no durable duplicate-create policy, generic operation registry,
  create manifest, cross-process fence, or unrelated Controller behavior.

[t368]: 368-wallet-startup-shutdown.md
[t453]: ../archive/tasks/453-serialize-manager-wallet-publication.md
