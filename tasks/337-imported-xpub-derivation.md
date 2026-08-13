# Task 337 — restore SQL imported-XPub public derivation

Component: **Wallet public derivation and Signer numeric adapter**.

Type: **Bug fix**.

Dependencies: **[Task 341][t341]**.

## Rationale

Store-routed `DerivePubKey` selects accounts by BIP44 account number, but an
arbitrary imported-xpub account intentionally has no public account number.
Public derivation therefore regresses even though its XPub is sufficient.

## Details

- Do not fabricate account zero or expose SQL row IDs as BIP44 numbers.
- Do not reinterpret kvdb internal account IDs as SQL account IDs.
- Preserve ADR 0013 normalized identity.
- Consume [Task 341][t341]'s wallet-owned selector containing scope and exactly
  one of account name or optional BIP44 account number. Run its validation
  before Store access; never accept an adapter or SQL row ID.
- Define an additive wallet-owned public derivation request containing that
  selector plus branch and child index, and expose it through a narrow public
  derivation interface implemented by `Wallet`. Do not add a method to the
  existing `Signer` interface and break external mocks merely to carry the new
  selector.
- Retain `Signer.DerivePubKey(ctx, BIP32Path)` as a checked numeric adapter. It
  constructs the numbered selector from `InternalAccount` and delegates to the
  selector-bearing implementation. Preserve the existing zero `Account` as
  unspecified; when `Account` is present, require it to be the matching hardened
  derivation child before Store access.
- Follow [Task 341][t341]'s reconciliation of the public selector with ADR
  0013; never expose the current kvdb `AccountID` projection.
- Private signing for an imported-xpub account must still fail.
- Land only the selector-bearing derivation request/interface and SQLite
  numberless-account derivation with collision and not-found coverage.
  [Task 329][t329] and [Task 376][t376] consume this task without waiting for
  modern-kvdb support.

## Suggested fix

Reuse the wallet-owned exactly-one semantic account selector from [Task 341][t341].
Resolve the account XPub through `db.Store`, then derive branch and index in
memory. Add a small selector-bearing public request and companion derivation
interface while keeping the existing `Signer` method as a numeric adapter.
Never expose a backend row identifier as the cross-backend selector.

## Verification

Locked public derivation works for imported-XPub accounts on SQLite, derived
accounts still resolve by BIP44 number, and collision tests cannot select the
wrong XPub. An external package can name and invoke the selector-bearing
request without importing internal DB types, while existing `Signer`
implementations remain source-compatible. The numeric adapter rejects
inconsistent paths before Store access. A private signing request for an
imported identity never falls back to a numbered derived account and performs
no secret lookup. An output-signing request for an imported-XPub child with no
derivation metadata fails before `GetAddressSecret` or any raw-import secret
path.

## Definition of done

- Task 341's wallet-owned selector and this task's public derivation request
  support numberless SQL imported-XPub accounts without exposing backend
  identifiers.
- Numeric derivation remains source-compatible and private signing never
  falls back to another account or secret path.
- Modern-kvdb acceptance is owned separately by
  [Task 361](361-kvdb-imported-xpub-derivation.md).

[t329]: 329-atomic-wallet-account-creation.md
[t341]: ../archive/tasks/341-public-role-contract.md
[t376]: 376-orchestrate-legacy-conversion.md
