# Task 421 — reconcile SQL passphrase commits

Component: **WalletVault and SQL wallet-secret persistence**.

Type: **Bug fix**.

Dependencies: **None**.

## Rationale

The shared SQL runtime classifies some commit failures as ambiguous, but
`WalletVault.ChangePassphrase` returns that error without determining whether
the old or new credential facts committed. Controller delivery cannot be
authoritative until the Vault settles that storage outcome.

## Details

- Before the write, retain only the exact persisted old verifier/ciphertext
  facts needed for comparison and the exact prepared new facts.
- On an ordinary pre-commit or definite rollback error, leave the current
  failure behavior unchanged.
- On `runtime.ErrAmbiguousTxCommit`, re-read the wallet-secret row under one
  bounded reconciliation context independent of the failed transaction.
- Classify the row as proven old only when its complete credential tuple
  matches the old snapshot, and proven new only when it matches the prepared
  new tuple. A same-passphrase rotation is decided by those persisted facts,
  not by testing both passphrases against a digest.
- Return success for proven new, a definitive internal unchanged/failure
  result for proven old, and a typed internal indeterminate result when the
  Store is unavailable or the row matches neither complete tuple. Do not keep
  advertising a reconciled-old outcome as an unclassified commit.
- Preserve the Vault's initial locked or unlocked state. Rotation and
  reconciliation do not replace or clear existing unlocked runtime keys.
- Keep serialization inside the existing WalletVault mutex. Do not add a
  durable generation, cross-Manager fencing, retries of the mutation, or a
  generic reconciliation framework.
- Zero every prepared master key, decrypted crypto key, and comparison buffer
  on every exit.

## Suggested fix

Factor one WalletVault-local comparison of persisted wallet-secret facts.
Snapshot the old tuple, prepare the new tuple, perform the current Store
update, and invoke the comparison only for an ambiguous commit. Return a
narrow keyvault-owned old/new/indeterminate result for Task 338 to translate.

## Verification

- Injected SQL outcomes prove definite rollback, committed-new,
  ambiguous-old, ambiguous-new, readable-neither, and unavailable cases on
  SQLite and PostgreSQL Store fixtures.
- A same-passphrase rotation is classified from exact row facts.
- Every test checks that locked/unlocked runtime state is unchanged and all
  prepared buffers are destroyed.
- No mutation retry, credential generation column, or cross-Manager lock is
  added.

## Definition of done

- An ambiguous SQL wallet-secret commit resolves to proven old, proven new,
  or one typed indeterminate result without guessing.
- WalletVault returns a definitive storage result to Task 338 and retains no
  prepared secret material.
- The PR is local to SQL passphrase durability and adds no Controller,
  lifecycle, or generic reconciliation machinery.
