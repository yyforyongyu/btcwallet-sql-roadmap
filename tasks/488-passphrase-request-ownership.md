# Task 488 — own passphrase request buffers through completion

Component: **Controller.ChangePassphrase request ownership**.

Type: **Bug fix**.

Dependencies: **[Task 331][t331]**.

## Rationale

`ChangePassphrase` can return on caller cancellation while its accepted handler
continues using the caller's passphrase slices. Clearing or reusing those
buffers after return can change the credentials used by the ongoing rotation.

## Details

- Keep caller-cancelable waiting. Document that cancellation after acceptance
  can leave the mutation outcome unknown and does not prove rollback.
- Give accepted work stable ownership of the selected passphrase inputs,
  preserving their values and nil/empty semantics after the caller returns.
  Clear operation-owned copies on completion and every rejected handoff.
- Keep the fix in the existing Controller request boundary in `package wallet`.
  Preserve caller buffers and the existing buffered response delivery.
- Exclude Vault/Store changes, a new result or error API, response deadlines,
  retries, and result registries. [Task 489][t489] owns shutdown context.

## Suggested fix

Copy the selected secret buffers before handing the request to the handler,
with clear ownership transfer and cleanup on both admission outcomes.

## Verification

- A controlled accepted call returns on caller cancellation; clearing or
  reusing its original buffers cannot change what the handler reads.
- Cover private and legacy public selections, valid empty public values, and
  owned-buffer clearing after success, failure, and rejected admission.
- Reuse existing pre-handoff cancellation and Vault classification tests.

## Definition of done

Returning from `ChangePassphrase` releases the caller's buffer-lifetime
obligation while accepted work retains stable inputs and clears its copies.
Caller cancellation remains supported with documented outcome uncertainty.

[t331]: ../archive/tasks/331-manager-wallet-lifecycle.md
[t489]: 489-passphrase-shutdown-context.md
