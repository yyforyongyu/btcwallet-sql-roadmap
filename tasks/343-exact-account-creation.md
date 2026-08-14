# Task 343 — create next-or-exact root-derived accounts

Component: **AccountManager root-derived creation and SQL AccountStore CAS**.

Type: **Gap fix**.

Dependencies: **[Task 332][t332], [Task 377][t377], [Task 388][t388],
[Task 392][t392], [Task 393][t393], [Task 394][t394], and [Task 398][t398]**.

## Rationale

The maintained account API cannot create a sparse exact account,
and a sequential invalid BIP32 child rolls back to the same cursor. lnd needs
exact purpose-1017 family numbers without filler accounts, but custom-scope and
external-XPub identity are independent extensions.

## Details

- Replace the positional root-derived call with the final wallet-owned
  `NewAccountParams` subset containing scope, optional exact account number,
  name, and discovery mode. Nil number preserves sequential creation.
- Limit this task to existing canonical scopes and wallet-root-derived account
  material. Reject a schema override or external account key until their
  follow-ups land.
- Validate every account component at or below the public maximum before adding
  the hardened bit or touching Vault/Store.
- For exact creation, require a vacant number and atomically update the cursor
  to checked `max(current, exact+1)`. Lower sparse holes remain available for
  later exact creation.
- Read and validate the expected account cursor, then use the maintained raw
  Vault decrypt/encrypt boundary and wallet derivation helpers to prepare the
  account key, both branches, ciphertext, and public facts outside the Store
  write. An invalid exact child leaves account and cursor unchanged.
- For sequential creation, skip and durably consume invalid account or branch
  children in one bounded CAS. Persist terminal exhaustion so retry cannot
  loop forever.
- Persist account, encrypted account secret, fingerprint, discovery mode, and
  cursor in one method-specific SQL transaction. Consume
  Task 388 in that same admission.
- On `runtime.ErrAmbiguousTxCommit`, return one wallet-owned indeterminate
  result and never retry from the transport error. Exact reconciliation is the
  next task.
- Do not create an initial live-watch obligation: an account row exposes no
  receiving script. Address allocation owns the first watchable script.
- On modern kvdb, preserve its existing sequential root-derived creation
  subset. Adapt only requests equivalent to that legacy operation: no exact
  number and no SQL-only field or semantic. Return a stable unsupported error
  before mutation for the exact form or another unsupported extension; do not
  extend the legacy format.
- Exclude custom scopes, supplied XPub accounts, arbitrary numberless import
  changes, Manager genesis (Task 329), and genesis secret ownership
  (Task 367).
- [Task 382](382-vault-account-derivation.md) later moves this preparation
  behind a typed Vault operation without changing allocation, invalid-child
  handling, CAS sequencing, or public behavior.

## Suggested fix

Prepare root-derived material with the maintained Vault before opening the
Store write, then commit one expected-cursor account CAS that handles
nil/exact allocation, skipped children, account facts, secrets, and discovery
mode. Keep the maintained kvdb adapter on its existing sequential operation
instead of sending it through the SQL CAS.

## Verification

SQLite and PostgreSQL cover sequential and sparse exact
creation, lower-hole exact fill, bounds, concurrent exact/sequential requests,
invalid account and branch children, exhaustion, mode selection, restart, and
ambiguous return without retry. Modern-kvdb fixtures preserve the existing
sequential subset and prove exact or SQL-only requests fail before Vault or
Store mutation. Spies prove derivation and Vault decrypt/encrypt work finish
before the Store write opens.

## Definition of done

Canonical SQL scopes support one final next-or-exact
root-derived operation with atomic account/secret/mode/cursor facts; invalid
children cannot wedge. Maintained modern kvdb retains its existing sequential
subset until Task 353 retires it; no custom, XPub, genesis, watch-delivery, or
generic reconciliation work is included.

[t332]: 332-watch-only-script-policy.md
[t377]: 377-account-manager-wallet-dto.md
[t388]: 388-runtime-account-xpub-collisions.md
[t392]: 392-discovery-mode-account-creation.md
[t393]: 393-key-only-receiving-rejection.md
[t394]: 394-key-only-transaction-ownership.md
[t398]: 398-semantic-account-recovery-targets.md
