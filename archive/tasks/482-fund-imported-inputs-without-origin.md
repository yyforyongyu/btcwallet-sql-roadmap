# Task 482 — fund imported inputs without known origin

Component: **PsbtManager imported-input decoration**.

Type: **Gap fix**.

Dependencies: **[Task 479][t479]**.

## Rationale

`FundPsbt` rejects an owned numberless imported-XPub input when its wallet
address has no derivation path. Lnd's raw TxTemplate supplies no input origin;
the Wallet must not fabricate a missing fingerprint.

## Details

- Fund and decorate supported owned imported inputs using verified UTXO and
  script facts, retaining truthful caller metadata when supplied.
- Omit unknown BIP32 origin; reject metadata that contradicts known Wallet
  facts. Preserve existing signing authorization and canonical change use.
- Exclude new API fields, invented account numbers, and Task 391's exact
  remote-signer paths.

## Suggested fix

Relax only the internal missing-origin branch of current PsbtManager
decoration, reusing the existing metadata validation and funding flow.

## Verification

Focused `FundPsbt` tests cover absent and caller-supplied origin plus a
contradictory key. Task 470 reruns unchanged imported-account PSBT itests after
Tasks 480/481 where those imports or scripts are needed. The
[published diagnostic PoC][wallet-poc] covers the existing API boundary.

## Definition of done

Owned imported inputs fund without invented origin while contradictory facts
still fail.

[t479]: ../archive/tasks/479-mix-imported-xpub-accounts.md

[wallet-poc]: https://github.com/yyforyongyu/btcwallet/commit/2c8d6196f731860925ce48f8f9bc68d1599e7b11
