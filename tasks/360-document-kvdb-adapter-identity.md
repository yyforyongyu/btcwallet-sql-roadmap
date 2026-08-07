# Task 360 — document kvdb adapter identity

Component: **ADR 0013 and the modern kvdb adapter contract**.

Dependencies: **None**.

## Rationale

The modern kvdb adapter needs a stable scope-local account identity for
recovery and lookup, but that backend detail is not a portable public account
ID. ADR 0013 must state the distinction before public wallet roles are changed.

## Details

- Amend ADR 0013 to document the stable scope-local kvdb adapter identity and
  the recovery and lookup behavior that requires it.
- State that the adapter identity remains internal and cannot be exposed as a
  cross-backend account ID, SQL row ID, or BIP44 account number.
- Preserve numberless imported-XPub identity and optional BIP44 account-number
  semantics in the public model.
- Define how internal recovery selectors retain kvdb identity without the
  exported `RecoveryState.AccountID` accessor.
- Produce only the ADR amendment. Wallet-owned DTOs, public errors,
  conversions, and role implementation belong to
  [Task 341](341-public-role-contract.md).

## Suggested fix

Update ADR 0013 with one explicit internal-adapter-identity section covering
scope locality, stability, recovery use, and public non-portability. Use
semantic wallet selectors for the public contract and leave backend IDs behind
the adapter boundary.

## Verification

- Compare the amendment with current kvdb recovery and lookup paths to ensure
  removing the internal identity would break behavior it claims to preserve.
- Confirm the documented public selector can represent numbered and
  numberless accounts without accepting an adapter or SQL row ID.
- Review the amendment independently of any role implementation change.

## Definition of done

- ADR 0013 explicitly preserves kvdb's internal scope-local identity while
  rejecting it as a portable public identifier.
- [Task 341](341-public-role-contract.md) can implement wallet-owned role types
  without another identity decision.
- The PR changes no public API, recovery implementation, or backend storage.
