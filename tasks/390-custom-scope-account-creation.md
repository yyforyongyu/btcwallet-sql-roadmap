# Task 390 — create root-derived accounts in custom scopes

Component: **AccountManager custom scope and SQL key-scope/account mutation**.

Type: **Gap fix**.

Dependencies: **[Task 343][t343], [Task 387][t387], and [Task 389][t389]**.

## Rationale

The Store can insert a new scope when given a schema, but public
derived-account creation never passes one and does not reject a conflicting
schema for an existing scope. lnd needs exact purpose-1017 accounts.

## Details

- Add optional explicit scope schema to the final `NewAccountParams` and require
  it when the scope is unknown.
- Accept only account-key-derivable P2PKH, P2SH-P2WPKH, P2WPKH, and key-path
  P2TR branch schemas. Reject script, anchor, unknown, and unsupported forms
  before Vault/Store access.
- Reject purpose or coin components at or above the hardened boundary before
  adding `HardenedKeyStart`.
- If the scope exists, require the supplied schema to match exactly; never
  silently ignore a conflicting override.
- Reuse retained Task 343 for nil/exact numbering, discovery mode, Vault
  preparation, collision admission, cursor CAS, and the account-specific
  reconciliation successor.
- Extend Task 389's completed operation-specific result reader with the custom
  scope's exact schema presence and equality facts. On an ambiguous commit,
  report success only for the complete custom-scope/account state and never
  retry or let Task 389 anticipate this task's new fields.
- Keep supplied XPub/path-bound accounts in the next task.
- Do not remove dead-looking key-scope columns in this PR.

## Suggested fix

Pass one validated explicit schema through the existing
`EnsureKeyScopeWithOps` path, add the missing existing-scope equality check,
then delegate account creation to the retained CAS.

## Verification

Purpose-1017 exact and sequential accounts survive reopen on
both dialects; unknown scope without schema, conflicting existing schema,
unsupported branch types, and hardened-overflow components fail before secret
or Store mutation. Ambiguous-commit fault injection covers complete, absent,
and conflicting custom-scope schema/account state without retry.

## Definition of done

Root-derived accounts can use a truthful custom scope
and explicit schema through the final generic API, without external XPub,
schema cleanup, or lnd-specific methods. Ambiguous results compare the exact
custom-scope schema introduced by this task.

[t343]: 343-exact-account-creation.md
[t387]: 387-account-creation-identity-adr.md
[t389]: 389-runtime-account-commit-reconciliation.md
