# Task 479 — mix imported XPub and signing accounts

Component: **AccountManager custody and Wallet results**.

Type: **Gap fix**.

Dependencies: **None**.

## Rationale

A signing lnd wallet must track another signer's XPub in the same wallet.
Maintained SQL ImportAccount currently rejects that mixed custody.

## Details

- Admit imported watch-only XPub accounts alongside local signing accounts
  through existing AccountManager APIs.
- Report imported accounts, addresses, and UTXOs as tracked but not locally
  signable; preserve local-account signing.
- Reconcile ADR 0012 with this one-wallet policy. Exclude raw key/script
  imports, per-account address schema, PSBT decoration, and a second wallet.

## Suggested fix

Reuse persisted account provenance and the existing UTXO spendability
classification; no new public API or SQL migration is indicated by the PoC.

## Verification

Check import, address allocation, ownership, spendability, reopen, and local
signing on SQLite and PostgreSQL. Task 470 verifies the unchanged imported
account lnd scenario. Local PoC: `2e2c6aa42f0a78fddd74f09fb3f0fed493b6526f`.

## Definition of done

One signing Wallet can retain both account custodians without treating an
external key as locally signable.
