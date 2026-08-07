# Task 362 — return a definitive kvdb passphrase result

Component: **Controller and LegacyWalletVault**.

Dependencies: **[Task 338](338-passphrase-result.md) and
[Task 352](352-kvdb-combined-passphrase-rotation.md)**.

## Rationale

The SQL Controller result contract in
[Task 338](338-passphrase-result.md) does not prove that a maintained modern
kvdb request reports the coherent result of its dual-credential rotation.

## Details

- Reuse [Task 338](338-passphrase-result.md)'s pre-handoff cancellation and
  accepted-request result latch for modern kvdb `ChangePassphrase`.
- Consume [Task 352](352-kvdb-combined-passphrase-rotation.md)'s atomic public
  and private credential result rather than adding another rotation path.
- Once accepted, run rotation and any bbolt commit reconciliation under the
  bounded handler-owned context and operation-specific completion that Wallet
  and Manager shutdown join.
- Publish exactly one buffered result; post-acceptance caller or Wallet
  cancellation cannot replace or suppress it.
- Reconcile an ambiguous bbolt commit as proven old, proven new, or a
  wallet-owned indeterminate result before reporting which credential is
  authoritative.
- Keep cross-Manager and multi-process credential fencing unsupported, and add
  no generic Controller operation registry.
- If [Task 353](353-retire-modern-kvdb-backend.md) lands first, archive this
  task rather than implementing behavior that retirement immediately removes.

## Suggested fix

Route the accepted kvdb handler through
[Task 352](352-kvdb-combined-passphrase-rotation.md)'s guarded rotation result
and [Task 338](338-passphrase-result.md)'s acceptance-latched response and
shutdown completion. Reconcile the exact verifier state after ambiguous
commit before sending the one result.

## Verification

- Barrier tests prove cancellation before handoff returns the context error,
  while cancellation or shutdown after acceptance joins and returns the
  handler result.
- Injected bbolt outcomes prove old, new, readable-neither, and unavailable
  reconciliation without claiming an unprovable passphrase.
- Concurrent rotations on one published Wallet admit one transition and give
  the loser a definitive result without changing live or durable key state.

## Definition of done

- Every accepted modern-kvdb passphrase request returns exactly one coherent
  success, failure, or typed indeterminate result.
- Shutdown joins the accepted operation before Vault lock or Store close.
- The PR adds no rotation implementation beyond
  [Task 352](352-kvdb-combined-passphrase-rotation.md) and no generic
  Controller lifecycle machinery.
