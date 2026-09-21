# Task 369 — reject repeated Wallet unlocks without resetting timeout

Component: **Controller**.

Type: **Test**.

Dependencies: **None**.

## Rationale

At `sql-wallet` revision `48e7f4b`, [Vault rejection tests][vault-tests] already
prove no reauthentication, and the [Controller][controller] changes its timer
only after Vault Unlock succeeds. Controller tests cover first unlock and
auto-lock separately; the missing proof is that repeated Unlock preserves the
original schedule, with a documented public rejection contract.

## Details

- Choose the existing rejection contract. While a Wallet is unlocked, every
  repeated Unlock returns an already-unlocked error without validating the new
  passphrase.
- A rejected repeated Unlock leaves the first successful Unlock's auto-lock
  schedule exactly unchanged. A new positive, zero/defaulted, or negative
  timeout cannot extend, shorten, disable, or re-enable that schedule.
- Preserve the existing serialized authentication/timer behavior. Return the
  already-unlocked result without changing the timer; no new coordinator,
  reauthentication operation, or lifecycle mechanism is required.
- Document this behavior on the public Controller/Wallet Unlock contract and
  prove it at the Controller boundary for both maintained backends. Preserve
  the Vault-level `ErrVaultUnlocked` behavior; do not invent a Vault
  reauthentication operation.
- Limit this task to repeated Unlock authentication and timer behavior. Exclude
  a new public error taxonomy, Task 338's ChangePassphrase result delivery,
  passphrase rotation, Unlock request cancellation redesign, Manager
  publication, lifecycle Start/Stop, and timer-framework cleanup.
- This remains a narrow Controller contract/documentation and regression task:
  existing production behavior does not make its missing coverage obsolete.
  Task 427 keeps its original passphrase-change and first-unlock timeout scope.

## Suggested fix

Document the caller-facing rejection contract and exercise the existing
Controller handler and timer. Reuse Vault rejection coverage for the absence
of reauthentication and secret reads; no new clock or timer API is needed.

## Verification

- Repeat Unlock with correct and wrong passphrases and shorter/longer positive,
  zero/defaulted, and negative timeouts. Every repeat returns the same
  already-unlocked result and preserves the original deadline or disabled
  schedule. Observe the public locked state at the original expiry.
- Run focused Controller cases and reuse the existing SQL and kvdb Vault
  rejection tests; add no duplicate secret-read or timer-call-count matrix.

## Definition of done

- Repeated Unlock has one documented rejection contract for both maintained
  backends.
- Rejection performs no passphrase validation and leaves the active auto-lock
  schedule unchanged for every timeout form.
- Focused Controller tests prove authentication and deadline behavior without
  redesigning ChangePassphrase, Vault APIs, lifecycle, or Manager publication.

[controller]: https://github.com/btcsuite/btcwallet/blob/48e7f4b7e0f3808b5022fd817d12c6a4361bb9c1/wallet/controller.go#L993
[vault-tests]: https://github.com/btcsuite/btcwallet/blob/48e7f4b7e0f3808b5022fd817d12c6a4361bb9c1/wallet/internal/keyvault/vault_lifecycle_test.go#L148
