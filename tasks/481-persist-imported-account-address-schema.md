# Task 481 — preserve imported account address schema

Component: **AccountManager and SQL account address schema**.

Type: **Bug fix**.

Dependencies: **None**.

## Rationale

For `ImportAccount` with strict BIP-49 schema, `NewAddress` and `FundPsbt`
use scope-level script types instead of the imported account's requested type.
Lnd then receives P2WPKH change where its imported account requires P2SH.
The hybrid BIP-49 path also fails address allocation.

## Details

- Persist and return each imported account's effective external and internal
  address schema on SQLite and PostgreSQL.
- Use that account schema for address allocation and PSBT change within the
  existing public APIs; preserve true scope and XPub identity.
- Exclude PSBT origin inference, new RPC fields, and Task 391's exact
  path-bound remote-signer account identity.

## Suggested fix

Add the minimal two-dialect account-schema migration and use account-level
types wherever the current path falls back to key-scope types.

## Verification

Focused public tests cover strict and hybrid BIP-49 import, address and change
script type, reopen, and existing BIP-84/BIP-86 behavior. Task 470 reruns the
unchanged lnd imported-account variants.

## Definition of done

Imported account address and change scripts retain their requested schema
through allocation, funding, and reopen.
