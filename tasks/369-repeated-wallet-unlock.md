# Task 369 — reject repeated Wallet unlocks without resetting timeout

Component: **Controller**.

Type: **Test**.

Dependencies: **None**.

## Rationale

Both maintained Vault implementations already reject Unlock while unlocked
without validating the supplied passphrase, and the Controller currently
changes its auto-lock timer only after Vault Unlock succeeds. The public Wallet
contract and Controller-level tests do not settle whether a repeated Unlock
reauthenticates or changes the existing deadline, leaving callers unable to
rely on the observed behavior.

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

Keep the existing serialized handler order explicit: call Vault Unlock, return
its already-unlocked failure unchanged, and mutate the timer only after a first
successful unlock. Update the caller-facing Unlock documentation and add
Controller tests around the existing Vault contract. Add only the narrowest
controller-local controllable timer seam if deterministic deadline observation
cannot be expressed with the existing timer; do not add a generic clock or
reauthentication API.

## Verification

- Unlock once with a controlled deadline, then repeat with both the correct and
  wrong passphrases. Both repeats must return the already-unlocked error, make
  no second persisted-secret/authentication read, and leave the Wallet
  unlocked until the original deadline.
- Repeat with a shorter positive timeout, a longer positive timeout, zero
  (default), and a negative timeout. A deterministic timer barrier must prove
  none changes the first deadline; the original expiry must still lock the
  Vault exactly once.
- Run the focused Controller and Vault cases with
  `make unit pkg=wallet case=TestControllerUnlockRepeated` and the package race
  variant, followed by `make unit pkg=wallet`.

## Definition of done

- Repeated Unlock has one documented rejection contract for both maintained
  backends.
- Rejection performs no passphrase validation and leaves the active auto-lock
  schedule unchanged for every timeout form.
- Focused Controller tests prove authentication and deadline behavior without
  redesigning ChangePassphrase, Vault APIs, lifecycle, or Manager publication.
