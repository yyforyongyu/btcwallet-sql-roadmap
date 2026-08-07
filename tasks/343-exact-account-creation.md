# Task 343 — support exact and custom account creation

Component: **AccountManager**.

Dependencies: **[Task 332][t332], [Task 337][t337], [Task 340][t340],
[Task 341][t341], [Task 345][t345], and [Task 354][t354], plus accepted ADR 0012
and ADR 0013 amendments covering scope storage and path-bound account identity**.

## Rationale

The public account role cannot create unknown custom scopes, sparse exact
accounts, or truthful path-bound external XPub accounts. Account provenance and
known derivation path are currently conflated.

## Details

- Keep arbitrary imported XPub accounts numberless.
- Define path-bound external XPub identity without marking it root-derived.
- Permit path-bound XPub-only accounts only in watch-only SQL wallets; a
  spendable wallet must provide matching private account material.
- Require the immutable [Task 345][t345] discovery mode in every creation request.
  Persist it atomically with the account and include it in ambiguous-commit
  reconciliation.
- Require explicit schema for a new custom scope and reject conflicts.
- Permit only account-key-derivable branch schemas: P2PKH, P2SH-P2WPKH,
  P2WPKH, and key-path P2TR. Reject script, anchor, unknown, and every other
  branch form before Store or Vault access.
- Reject purpose or coin components at or above the hardened boundary before
  adding `HardenedKeyStart`.
- Require a path-bound XPub's hardened child number to equal the requested exact
  account number before mutation.
- Reject an exact account number above `db.MaxAccountNumber` before any Store or
  Vault access, including before adding `HardenedKeyStart`.
- Advance `next_account_number` with checked `max(current, exact+1)` and leave
  lower sparse holes available for later exact creation.
- For sequential creation, atomically consume an account number whose account
  key or external/internal branch derivation returns `ErrInvalidChild`, then try
  the next number within the same bounded operation. An invalid exact number
  returns a typed error with no account or cursor mutation. Persist terminal
  exhaustion at `db.MaxAccountNumber` so retries cannot loop forever.
- For root-derived accounts, compute and persist a present master fingerprint;
  if the caller supplies one, require exact equality. For external XPub
  accounts, preserve absent versus present-zero exactly and never invent a root
  fingerprint.
- Normalize the account XPub payload once and compare it wallet-wide in the same
  creation CAS. Reject equal payloads whenever effective external or internal
  branch schemas overlap, including across scopes and before any address exists.
  Names, version bytes, optional numbers, provenance, or fingerprints do not
  make colliding child-key identities distinct. Serialize the conditional check
  and insert with the backend's account-creation transaction rather than waiting
  for a later child-script collision.
- Reuse [Task 337][t337]'s durable selector and [Task 341][t341]'s public account types.
- Resolve and derive private account material through [Task 354][t354]'s typed Vault
  operation; account creation must not receive encrypted rows or retain
  plaintext keys outside the Vault callback.
- Treat the backend-local SQL CAS result as potentially ambiguous. Re-read the
  exact scope schema, account identity and material, discovery mode,
  fingerprint, and next-account cursor. Return committed, no-mutation failure,
  or a wallet-owned indeterminate result; never retry from only the transport
  error.
- For a `watch-chain` account, atomically enqueue initial external and internal
  branch-horizon obligations with account creation. Return one typed pending
  result and expose no receiving-ready account until the Syncer completes both
  live-tip registrations. A key-only account creates no chain obligation and
  is returned only through non-receiving account APIs.
- On the maintained kvdb Manager, return a stable wallet-owned unsupported
  sentinel before mutation for exact, custom-scope, and path-bound requests
  that waddrmgr cannot represent truthfully. Do not expand the legacy format.
- Remove dead `coin_pub_key` and `key_scope_secrets` schema while touching scope
  contracts only after amending both ADR 0012 and ADR 0013; ADR 0012 currently
  retains `key_scope_secrets` explicitly.

## Potential fix

Update ADRs 0012 and 0013, then add one generic account request with an optional
exact number, external XPub, schema, and fingerprint. Validate path components
with checked arithmetic, prepare derivation outside the write, and commit
scope/account/counter changes by backend-local CAS. Use a callback-scoped Vault
derivation operation that zeros private material on every exit. Keep
initial-account transactionality in [Task 329][t329]. Reconcile exact durable facts
after an ambiguous commit without adding a generic account manifest. Reuse
[Task 345][t345]'s ordered branch-horizon delivery for watch-chain accounts.

## Done when

Custom scopes and sparse exact accounts survive restart, invalid combinations
and over-maximum account numbers fail before dependency access, provenance
remains truthful, and no public type exposes internal DB identity. Ambiguous
commit tests prove exact committed, absent, and indeterminate outcomes without
duplicating an account or advancing its cursor twice. Discovery mode and
fingerprint semantics survive reopen, and unsupported kvdb shapes fail before
mutation. Invalid sequential children are durably consumed, invalid exact
children leave no mutation, both branches are validated, and terminal exhaustion
is stable. Watch-chain results remain pending until both initial horizons are
registered; unsupported schema branches fail before dependency access.
Concurrent imported-XPub and root-derived duplicate-identity tests admit one
account and return the typed collision for the other without advancing the
cursor, creating watch obligations, or relying on address allocation to expose
the conflict. A zero-counter BIP49Plus/BIP84 pair with equal normalized payloads
is rejected when an overlapping branch would derive the same scripts.

[t329]: 329-atomic-wallet-account-creation.md
[t332]: 332-watch-only-script-policy.md
[t337]: 337-imported-xpub-derivation.md
[t340]: 340-durable-database-identity.md
[t341]: 341-public-role-contract.md
[t345]: 345-durable-watch-registration.md
[t354]: 354-vault-owned-secret-resolution.md
