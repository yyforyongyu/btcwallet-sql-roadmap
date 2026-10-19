# Task 390 — create root-derived accounts in custom scopes

Component: **AccountManager custom scope and SQL key-scope/account mutation**.

Type: **Gap fix**.

Dependencies: **[Task 343][t343], [Task 387][t387], and
[Task 468][t468]**.

## Rationale

The Store can persist a new key scope when it receives a schema, but the
maintained account-creation API cannot supply that schema or reject a
conflicting schema for an existing scope. Lnd therefore cannot create its
purpose-1017 key-family accounts through the generic Wallet API.

## Details

- Add an optional scope schema to [Task 392][t392]'s `NewAccountParams`; do not
  add a key-family-specific method or another request type.
- Require both a schema and an exact account number when creating an account
  in an unknown scope. Use Task 343 for exact-number creation and Task 468 for
  `NoChainSync=true` activation.
- Accept only supported account-key branch schemas and reject invalid,
  hardened-overflow, or conflicting scope requests before secret preparation
  or Store mutation.
- If the scope already exists, require any supplied schema to match it.
- Return the public errors established by [Task 378][t378]. Modern kvdb remains
  unsupported for new custom-scope creation.
- Keep supplied-XPub accounts, sequential invalid-child repair, and exact
  ambiguous-commit reconciliation in their existing tasks.
- Keep production changes within AccountManager and its wallet-owned Store
  boundary. Do not add lnd-specific APIs or a second scope registry.

## Suggested fix

Validate the optional schema at AccountManager and pass it through the
existing key-scope mutation before delegating exact account creation to Task
343. An equivalent local implementation is acceptable if it preserves the
same public contract and mutation ordering.

## Verification

- On SQLite and PostgreSQL, create and reopen an exact purpose-1017 account
  with both `NoChainSync` values.
- Prove missing, unsupported, overflowing, and conflicting schemas fail before
  secret or Store mutation with the expected public error identity.
- Prove modern kvdb rejects the operation before secret preparation or Store
  mutation.
- Prove an ambiguous mutation is not retried and does not publish runtime
  account state.

## Definition of done

The generic account-creation request can create an exact root-derived account
in a validated custom scope without a purpose-specific API or new component.

[t343]: 343-exact-account-creation.md
[t378]: 378-account-manager-error-boundary.md
[t387]: 387-account-creation-identity-adr.md
[t392]: 392-discovery-mode-account-creation.md
[t468]: 468-activate-no-chain-sync-account-creation.md
