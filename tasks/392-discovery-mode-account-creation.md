# Task 392 — consume discovery mode in existing account creation

Component: **AccountManager and SQL AccountStore create inputs**.

Type: **Gap fix**.

Dependencies: **[Task 345][t345]**.

## Rationale

The schema migration alone does not ensure every existing
creation path supplies the durable fact. Current sequential and imported-XPub
operations must remain truthful before the exact-account API extends them.

## Details

- Require a discovery mode in internal SQL derived/imported account create
  parameters and carry it through returned account information.
- Make the existing positional `NewAccount` and ordinary numberless
  `ImportAccount` compatibility paths explicitly create `watch-chain`
  accounts. Do not infer mode from purpose, address type, or caller backend.
- Reject a missing or per-call mode override before secret preparation or
  Store mutation.
- Leave Manager genesis creation to Task 329 and configurable key-only account
  selection to retained Task 343.
- Keep modern kvdb format changes out of scope.

## Suggested fix

Thread the retained Task 345 enum through the existing
method-specific create parameters and hard-code the legacy public entry points
to their current receiving behavior.

## Verification

SQLite and PostgreSQL sequential/imported creation reopen as
`watch-chain`; invalid modes fail before Vault/Store mutation; existing APIs do
not create key-only accounts; kvdb storage is unchanged.

## Definition of done

Every pre-existing SQL account-creation path writes an
explicit truthful mode, while configurable selection, genesis, allocation,
ingestion, and live-watch delivery remain outside the PR.

[t345]: 345-durable-watch-registration.md
