# Task 360 — document kvdb adapter identity

Component: **ADR 0013 and the modern kvdb adapter contract**.

Type: **Design**.

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
- Produce only the ADR amendment. Semantic public identity belongs to
  [Task 341](341-public-role-contract.md), AccountManager result conversion
  belongs to [Task 377](../../tasks/377-account-manager-wallet-dto.md), and the
  private recovery adapter belongs to
  [Task 381](../../tasks/381-recovery-adapter-identity.md).
  The independent public error boundaries remain in Tasks 378–380.

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
- Tasks 341, 377, and 381 can implement their distinct identity, result, and
  private-adapter contracts without another identity decision.
- The PR changes no public API, recovery implementation, or backend storage.
