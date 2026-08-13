# Task 352 — rotate kvdb passphrases atomically

Component: **LegacyWalletVault and waddrmgr passphrase rotation**.

Type: **Bug fix**.

Dependencies: **None**.

## Rationale

The modern kvdb Vault rotates public and private passphrases with two
stateful manager calls inside one database transaction. A second-half or
commit failure can leave the Manager's live keys changed while the durable
bucket retains the old credential facts.

## Details

- Keep the legacy dual-encryption format and one-wallet limit.
- Prepare and validate every requested credential half before changing
  persistent or live key state.
- Give waddrmgr one operation-specific rotation guard and plan spanning
  preparation, the single `walletdb.Update`, commit reconciliation, and final
  live-key installation.
- Establish and document one guard-before-walletdb order for Vault Lock,
  Unlock, Encrypt, Decrypt, and rotation. Mechanically adapt only paths needed
  to obey that order; do not add a Store-wide write lock.
- Apply public and private row changes in one transaction and publish deep
  copies to Manager-owned live storage only after a successful or
  reconciled-new commit.
- Preserve the initial locked/unlocked state. A locked Manager retains only
  the public material its format requires; an unlocked Manager remains usable
  only under the new credentials after success.
- Reconcile a bbolt commit error against the exact old and prepared-new
  verifier facts. Return a typed indeterminate result if neither state can be
  proved.
- Zero every plan-owned master key, crypto key, and temporary buffer after
  transfer or abort.
- Exclude modern-kvdb account derivation, cursor CAS, invalid-child handling,
  and moving encryption outside Store transactions.
  [Task 343][t343] owns account-creation preparation and the cursor CAS.
  [Task 382][t382] later replaces only that raw preparation with a typed Vault
  operation; it does not take ownership of allocation, invalid-child, or CAS
  semantics.
- If Task 353 retires modern kvdb first, archive this task instead of
  implementing it.

## Suggested fix

Add a narrow waddrmgr rotation plan that derives both selected halves under
one operation guard without mutating live state, writes both halves in one
walletdb transaction, reconciles only an ambiguous commit, installs deep
copies for proven-new, and destroys the plan on every exit.

## Verification

- Injected failures before either write, between writes, and at commit prove
  durable and live state remain on the same old or new credential tuple.
- Ambiguous commit fixtures prove old, new, neither, and unavailable outcomes.
- Concurrent Lock, Unlock, Encrypt, Decrypt, and rotation obey one lock order
  without observing an intermediate credential half.
- Locked and unlocked rotations preserve their state, and plan zeroing cannot
  corrupt installed Manager-owned keys.
- No account cursor, derivation, invalid-child, or exact-account behavior
  changes.

## Definition of done

- Requested legacy credential halves rotate as one proven disk-and-memory
  result while preserving initial lock state.
- Every prepared secret is destroyed and every conflicting Vault operation
  follows the documented order.
- The PR adds no account-creation redesign, new kvdb format, or deprecated RPC
  behavior.

[t343]: 343-exact-account-creation.md
[t382]: 382-vault-account-derivation.md
