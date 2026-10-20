# Task 388 — reject colliding runtime account XPub identities

Component: **AccountStore account-creation admission**.

Type: **Bug fix**.

Dependencies: **[Task 341][t341], [Task 343][t343], [Task 378][t378], and
[Task 387][t387]**.

## Rationale

Task 365 validates only one Manager genesis request. Runtime
`ImportAccount` and root/custom account operations can still insert equal
normalized XPub payloads whose effective branch schemas derive the same
scripts.

## Details

- Normalize the account XPub payload independently of version bytes, names,
  optional numbers, provenance, and fingerprints.
- Reject equal payloads whenever external or internal effective branch schemas
  overlap, within or across scopes and before any child exists.
- Serialize the wallet-wide check with each account insert so concurrent
  requests admit at most one conflicting identity. Use the existing SQL
  transaction boundary on SQLite and PostgreSQL and the existing walletdb
  write transaction on modern kvdb; do not add a generic lock.
- Preserve non-overlapping equal payloads only when the accepted ADR permits
  their branch schemas to derive disjoint scripts.
- Apply the admission to existing ordinary `ImportAccount` on SQLite,
  PostgreSQL, and modern kvdb and define its wallet-owned
  `ErrAccountIdentityCollision` result here, where the collision first becomes
  callable. Apply the same admission to later runtime account-creation
  operations when their backend supports them. Task 365 remains the genesis
  preflight owner.
- After lifecycle and request validation, a duplicate name wins with
  `ErrAccountAlreadyExists`; check normalized-XPub overlap only for an
  otherwise-vacant name. Database constraint or lookup order must not change
  that precedence. Task 343's exact-number field is already implemented;
  preserve occupied-number-before-collision precedence on supported requests.

## Suggested fix

Add one method-specific normalized identity query and conditional insert check
to the existing SQL and modern-kvdb account workflows; do not wait for a later
address uniqueness failure or add a generic lock.

## Verification

SQLite, PostgreSQL, and modern kvdb cover same-scope and cross-scope equal
payloads, BIP49Plus/BIP84 overlap, non-overlap, and barrier-controlled
concurrent creation. Repeated requests that also occupy a name deterministically
match `ErrAccountAlreadyExists`; an otherwise-vacant loser creates no account,
cursor change, secret, or watch fact and matches
`ErrAccountIdentityCollision`. Existing exact-number requests also prove an
occupied number wins before collision classification.

## Definition of done

Runtime account creation on SQLite, PostgreSQL, and modern kvdb admits no
overlapping normalized XPub identity and returns the deterministic public
classification without changing Manager genesis validation.

[t341]: 341-public-role-contract.md
[t343]: 343-exact-account-creation.md
[t378]: 378-account-manager-error-boundary.md
[t387]: 387-account-creation-identity-adr.md
