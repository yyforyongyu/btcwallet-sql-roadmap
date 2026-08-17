# Task 392 — consume discovery mode in existing account creation

Component: **AccountManager and SQL AccountStore create inputs**.

Type: **Gap fix**.

Dependencies: **[Task 345][t345] and [Task 387][t387]**.

## Rationale

The Store contract alone does not give maintained callers final account-
creation request shapes. Sequential and imported-XPub operations must state
their discovery mode explicitly before exact-account behavior extends them.

## Details

- Consume Task 387's final wallet-owned account-creation parameters. Replace
  positional `NewAccount` and ordinary numberless `ImportAccount` inputs with
  their final request shapes, update maintained callers and mocks atomically,
  and remove the old signatures rather than retaining adapters.
- Require an explicit discovery mode and carry it through returned account
  information. Ordinary sequential and numberless-import requests admit only
  `watch-chain` until Task 343 enables its accepted configurable behavior. Do
  not infer mode from purpose, address type, or caller backend.
- Reject a missing mode or unsupported key-only selection before secret
  preparation or Store mutation.
- Leave Manager genesis creation to Task 329 and configurable key-only account
  selection to retained Task 343.
- Keep modern kvdb format changes out of scope.

## Suggested fix

Thread Task 345's enum through the final request types and Store inputs. Update
all maintained callers to construct those requests directly, reject unsupported
mode selection before mutation, and delete the positional signatures.

## Verification

SQLite and PostgreSQL sequential/imported creation through the final requests
reopens as `watch-chain`; invalid modes fail before Vault/Store mutation; no
old signature or legacy adapter remains; kvdb storage is unchanged.

## Definition of done

Every pre-existing SQL account-creation path writes an
explicit truthful mode through the final request shapes, while configurable
selection, genesis, allocation, ingestion, and live-watch delivery remain
outside the PR.

[t345]: 345-durable-watch-registration.md
[t387]: 387-account-creation-identity-adr.md
