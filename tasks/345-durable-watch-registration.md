# Task 345 — persist SQL account discovery mode

Component: **SQL account schema and AccountStore persistence**.

Type: **Gap fix**.

Dependencies: **[Task 340][t340]**.

## Rationale

SQL has no durable distinction between receiving accounts and
key-only locator accounts. The storage fact must exist, backfill safely, and
remain immutable before any caller can select key-only behavior.

## Details

- Define the wallet/store `watch-chain` and `key-only` enum with checked
  conversion at the Store boundary.
- Add the field in the SQLite and PostgreSQL schema, backfill every existing
  account as `watch-chain`, then make it required and immutable.
- Include discovery mode in account reads and method-specific create inputs.
  Reject unknown or changed modes at the Store boundary. Until Task 392 adapts
  every existing derived/imported caller, preserve its omitted-mode input as a
  narrow compatibility form that writes `watch-chain`; do not expose that
  default to new or configurable creation entry points.
- Treat accountless raw imports as implicitly `watch-chain`; do not invent an
  account row or a key-only raw-import form.
- Keep public selection policy and wallet consumers in the three follow-ups
  below.

## Suggested fix

Land one SQL migration and regenerated query set for both
dialects, plus the smallest Store enum/conversion and reopen conformance. Keep
the old derived/imported create-input adapter working by translating its
omitted mode to `watch-chain` until Task 392 removes that adapter. Make the
migration itself the only mutation owner.

## Verification

Migration fixtures prove old accounts become `watch-chain`, explicit new rows
require a valid mode, the retained caller shape still creates `watch-chain`,
updates cannot change mode, and reopen returns the exact mode on SQLite and
PostgreSQL. Generated output matches source SQL.

## Definition of done

Every SQL account has one immutable, validated discovery mode after migration;
existing derived/imported creation remains compatible until Task 392 makes its
mode explicit. No public selection behavior, delivery row, chain call, or
transaction-ownership change is included.

[t340]: 340-durable-database-identity.md
