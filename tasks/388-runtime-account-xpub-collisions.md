# Task 388 — reject colliding runtime account XPub identities

Component: **SQL AccountStore account-creation admission**.

Dependencies: **[Task 341][t341] and [Task 387][t387]**.

## Rationale

Task 365 validates only one Manager genesis request. Runtime
`ImportAccount` and later root/custom/path operations can still insert equal
normalized XPub payloads whose effective branch schemas derive the same
scripts.

## Details

- Normalize the account XPub payload independently of version bytes, names,
  optional numbers, provenance, and fingerprints.
- Reject equal payloads whenever external or internal effective branch schemas
  overlap, within or across scopes and before any child exists.
- Serialize the wallet-wide check with each SQL account insert so concurrent
  requests admit at most one conflicting identity.
- Preserve non-overlapping equal payloads only when the accepted ADR permits
  their branch schemas to derive disjoint scripts.
- Apply the admission to existing ordinary `ImportAccount` and expose a stable
  wallet-owned collision error. Task 365 remains the genesis preflight owner.

## Suggested fix

Add one method-specific normalized identity query and
conditional insert check to the shared SQL account workflow; do not wait for a
later address uniqueness failure or add a generic lock.

## Verification

Both dialects cover same-scope and cross-scope equal payloads,
BIP49Plus/BIP84 overlap, non-overlap, and barrier-controlled concurrent
creation. The loser creates no account, cursor change, secret, or watch fact.

## Definition of done

Runtime SQL account creation admits no overlapping
normalized XPub identity and returns one public collision classification
without changing Manager genesis validation.

[t341]: 341-public-role-contract.md
[t387]: 387-account-creation-identity-adr.md
