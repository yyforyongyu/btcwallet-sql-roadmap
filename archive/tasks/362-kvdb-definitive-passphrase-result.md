# Task 362 — return a definitive kvdb passphrase result

Component: **Controller and LegacyWalletVault**.

Type: **Bug fix**.

Dependencies: **[Task 338](338-passphrase-result.md) and
[Task 352](352-kvdb-combined-passphrase-rotation.md)**.

> **Superseded 2026-09-18:** SQL and kvdb share Controller.ChangePassphrase,
> sendReq, mainLoop, and the Vault call at sql-wallet 2ae306dea8c5. Eliminate
> the separate backend adaptation layer: Task 338 owns the single shared
> accepted-response fix, while Task 352 retains kvdb uncertain-rotation
> containment. No second delivery implementation is needed.

> Follow-up 2026-10-20: see archived [Task 338](338-passphrase-result.md) for
> its later supersession and replacement policy; Task 352 remains deferred.

## Rationale

The maintained modern-kvdb `Controller.ChangePassphrase` can accept a request
and settle its Vault operation after caller cancellation or Wallet shutdown
has already won the response wait. The caller then receives a transport result
instead of Task 352's exact success, determinate failure, or
`ErrWalletReopenRequired` result.

## Details

- Reuse [Task 338](338-passphrase-result.md)'s pre-handoff cancellation,
  buffered accepted-request result, bounded handler context, and
  operation-specific completion for modern kvdb `ChangePassphrase`.
- Once accepted, run the existing `LegacyWalletVault.ChangePassphrase` under
  the handler-owned context and publish exactly one result. Caller or Wallet
  cancellation after acceptance cannot replace or suppress it.
- Return Task 352's success, determinate pre-mutation error, or
  `ErrWalletReopenRequired` result unchanged at the public Controller boundary
  with its `errors.Is` identity. Do not reinterpret the terminal result as
  proof of an old or new durable credential.
- Join the accepted handler before private Wallet stop calls the permitted
  Vault Lock. A reopen-required caller must complete Manager Stop before
  constructing a fresh Manager and selecting the reopened Wallet from Start.
- Do not add another rotation path, retry a credential mutation, probe both
  passphrases while the old Manager is live, or bypass Task 352's terminal
  latch.
- Keep cross-Manager and multi-process credential fencing unsupported, and add
  no generic Controller operation registry.
- Modern kvdb remains maintained for the current and next release. Deferred
  retirement does not eliminate this correction.

## Suggested fix

Route the accepted kvdb handler through Task 338's acceptance-latched response
and shutdown completion, and forward Task 352's exact classified Vault result
once. Keep durable recovery caller-driven through Manager Stop and a fresh
Manager Start instead of adding live-manager reconciliation.

## Verification

- Barrier tests prove cancellation before handoff returns the context error,
  while cancellation or shutdown after acceptance joins and returns the
  handler result.
- Inject a determinate pre-mutation failure, successful combined rotation, and
  each post-mutation second-half or commit failure. Every caller receives the
  exact Task 352 result once.
- For a reopen-required result, prove later credential calls reach no legacy
  manager method, Manager Stop still locks the Vault, and fresh Manager Start
  reads the durable credential state.
- Concurrent rotations on one published Wallet serialize through Task 352's
  guard and give every caller one classified result without a second mutation
  path.

## Definition of done

- Every accepted modern-kvdb passphrase request returns exactly one success,
  determinate failure, or `ErrWalletReopenRequired` result from Task 352.
- Shutdown joins accepted passphrase work before Vault Lock and Store close,
  and terminal recovery uses a fresh Manager rather than the uncertain one.
- The PR adds no rotation implementation beyond Task 352, live credential
  inference, generic Controller lifecycle machinery, or excluded legacy
  production change.
