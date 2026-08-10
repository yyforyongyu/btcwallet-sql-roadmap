# Task 331 — fence Manager admission and Store closure

Component: **Manager**.

Dependencies: **[Task 368][t368]**.

## Rationale

Concurrent same-name loads can both miss the cache, open resources, and
overwrite the published Wallet. The legacy one-wallet kvdb path can also race
across different requested names. `Manager.Close` currently closes the shared
backend directly, without preventing new Create, Load, or retained-Wallet
Start work, joining admitted operations, or stopping published Wallets first.

## Details

- Define one public wallet-owned `ErrManagerClosed` sentinel for Create, Load,
  and a retained managed Wallet's Start after Manager closure.
- Give the Manager one lifecycle admission owner. Before backend work, Create
  and Load either join the admitted-operation count or return
  `ErrManagerClosed`; admission cannot be added after Close marks the Manager
  closed.
- Serialize build and publication for each SQL wallet name so concurrent Loads
  return one published Wallet and no losing assembly leaks resources. Serialize
  the entire legacy one-wallet kvdb Create/Load admission path across requested
  names.
- Keep durable duplicate-create authority and `ErrWalletExists` in Task 330.
  This task supplies only the in-memory admission and publication ordering that
  Task 330 consumes as a fast path.
- Make one Close owner atomically close admission, revoke the Manager-owned
  startup lease, wait for every already-admitted Create or Load to publish or
  fail, stop and join every published Wallet, and only then close the shared
  backend. Concurrent or repeated Close calls join that completion and return
  its result rather than closing the backend again.
- Construct every managed Wallet with the same Manager-owned startup lease.
  Start checks it before beginning startup and holds its gate across the final
  cancellation check, every goroutine publication, and the transition to
  `Started`. Close revokes under that gate and releases it before calling Stop.
  If Start owns the gate first, it publishes one complete generation that Close
  then stops; if Close owns it first, Start publishes nothing and returns
  `ErrManagerClosed`.
- Use the lock order `Manager admission gate -> startup lease gate` when both
  are needed. Never wait for admitted operations or Wallet completion while
  holding either gate. Wallet lifecycle synchronization is entered from Start
  under the lease gate; Close releases the lease gate before entering Stop, so
  it never inverts that order.
- Preserve Manager ownership of the Store and restartable Wallet Stop. Do not
  evict Wallets or make a stopped Wallet terminal.
- Exclude Task 358's context parameters and cancellation cutoff, Task 330's
  durable duplicate classification, Task 338's accepted passphrase operation,
  repeated Unlock behavior, arbitrary non-lifecycle Wallet call fencing,
  generic operation registries, publication state machines, helper chains,
  create manifests, and cross-process Manager fencing.

## Suggested fix

Add a small Manager lifecycle gate containing the closed bit, admitted-operation
completion, per-name publication guards, and one Close completion/result. Use
the closed-bit lock to serialize `Add` against Close before releasing callers
into backend work. Clean up a per-name guard on both publication and failure;
do not add `awaitCreate`, eviction callbacks, or a publication state machine.

Pass one small shared startup-lease object into every Wallet assembled by the
Manager. Compose its final Start gate with Task 368's one
pre-publication lifecycle critical section. Close first prevents new admission
and revokes the lease, then waits without either lock, calls Stop on the stable
published-Wallet snapshot, and closes the backend only after every Stop joins
its completion.

## Verification

- Add a barrier-controlled same-name concurrent Load test that proves exactly
  one Wallet is assembled and published and every successful caller receives
  that pointer.
- Add a kvdb cross-name Create/Load admission race proving the deprecated
  backend never admits two live wallet assemblies.
- Race Close against Create and Load on both maintained backend paths. Work
  admitted before the close point must settle before backend close; later work
  must return `ErrManagerClosed`; no Wallet may publish after backend close.
- Use an ordered fake backend and Wallet completion barriers to prove Close
  invokes backend close only after all admitted operations and all published
  Wallet Stops finish. Concurrent Close calls must join one close and return
  the same result.
- Race Close against Start with barriers immediately before and after the final
  lease check. Revocation must be indivisible from goroutine publication and
  `Started`: either one complete started generation is subsequently stopped or
  no generation is published. A post-Close Start must return
  `ErrManagerClosed` without Store access.
- Run the focused Manager race cases with
  `make unit-race pkg=wallet case=TestManager...`, followed by
  `make unit pkg=wallet`.

## Definition of done

- Create and Load have one library-owned admission and publication order for
  SQL names and the whole legacy kvdb path.
- Close rejects later admission, joins prior admission, prevents retained
  Wallet startup, joins every published Wallet, and closes the shared backend
  last and exactly once.
- No lifecycle race publishes against a closed Store or leaks a losing Wallet
  assembly.
- The PR adds no durable duplicate-create policy, context API, terminal Wallet
  state, eviction callback, publication state machine, generic registry, or
  cross-process fencing.

[t368]: 368-wallet-startup-shutdown.md
