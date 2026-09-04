# Task 381 — hide recovery adapter identity

Component: **RecoveryState public contract and internal Syncer adapter**.

Type: **Refactor**.

Dependencies: **[Task 360][t360]**.

> **Superseded 2026-09-05:** No maintained public consumer requires this
> identity-only refactor. The roadmap retains the current shape until a
> concrete public contract demonstrates a need to change it.

## Rationale

`RecoveryState.AccountID` exports SQL row or kvdb adapter identity even though
only wallet-internal Syncer code currently consumes it. Task 360 preserves the
identity for internal recovery and lookup, not as portable public state.

## Details

- Remove the exported `RecoveryState.AccountID` accessor.
- Keep the accepted Task 360 scope-local adapter identity inside recovery
  state and expose it only through a wallet-private selector or helper used by
  Syncer horizon construction.
- Resolve numberless imported-XPub recovery through the semantic account name
  without equating BIP44 numbers, SQL row IDs, and kvdb adapter IDs. Remove any
  obsolete public fallback shape.
- Do not change scan targets, scan budgeting, horizon persistence, recovery
  window validation, or ADR identity policy.
- Keep public semantic selector types in Task 341 and recovery scan work in
  Task 363.

## Suggested fix

Replace the exported accessor with an unexported recovery selector carrying
the backend identity only as private adapter data. Update the internal Syncer
consumer directly and remove any public documentation that describes the ID as
portable.

## Verification

- SQL and modern-kvdb recovery tests resolve every numbered and numberless
  account and persist the same horizons after the accessor removal.
- The public recovery itest uses `RecoveryState` through the imported `wallet`
  package without an adapter-ID API, and public API inspection finds no account
  row or adapter identifier.
- Collision fixtures with equal per-scope kvdb adapter numbers remain isolated
  by scope internally.

## Definition of done

- Recovery retains required backend identity privately and exports none of it.
- Syncer behavior is unchanged for SQL and modern kvdb.
- No scan algorithm, window, DTO, or public selector behavior is included.

[t360]: 360-document-kvdb-adapter-identity.md
