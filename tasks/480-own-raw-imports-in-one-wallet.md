# Task 480 — own raw imports in one signing wallet

Component: **AddressManager raw imports and SQL address ownership**.

Type: **Gap fix**.

Dependencies: **[Task 479][t479] and [Task 403][t403]**.

## Rationale

Lnd's ImportPublicKey and ImportTapscript track externally controlled outputs.
A signing SQL wallet currently rejects those imports, and raw address rows
have no truthful account ownership under the visible `imported` alias.

## Details

- Store public-key and tapscript imports under a watch-only import account in
  the same Wallet; preserve account balance, UTXO, and transaction reporting.
- Persist ownership without inventing an HD branch or child index. Add the
  minimal SQL migration and query changes needed for both dialects.
- Amend ADR 0012 for raw public-only and script-only imports in signing SQL
  wallets while preserving the rule that scripts do not grant signing power.
- Preserve existing lnd WalletKit RPC requests and responses. Task 403 owns
  watch registration; Task 482 owns PSBT metadata.

## Suggested fix

Extend Wallet-owned import admission and address ownership using the existing
account and address stores, not a second Wallet or RPC account selector.

## Verification

Prove imported output tracking and reopen through public Wallet results on
SQLite and PostgreSQL. Task 470 reruns existing public-key and tapscript lnd
itests, including cooperative-close delivery.

## Definition of done

Raw public-only imports remain visible under `imported` in a signing Wallet
without a fabricated derivation path, and ADR 0012 states this policy.

[t479]: ../archive/tasks/479-mix-imported-xpub-accounts.md
[t403]: ../archive/tasks/403-raw-import-watch-obligations.md
