# Task 391 — create exact path-bound XPub accounts

Component: **AccountManager and SQL account, address, and UTXO identity
shape**.

Type: **Gap fix**.

Dependencies: **[Task 332][t332], [Task 337][t337], [Task 378][t378],
[Task 387][t387], [Task 388][t388], [Task 390][t390],
[Task 392][t392], and [Task 395][t395]**.

## Rationale

A uniformly watch-only wallet needs exact depth-three account XPub
installation without pretending the XPub came from its wallet root or
collapsing a known account number into arbitrary numberless import identity.
Lnd's online remote-signer setup imports accounts with explicit scope/number
identity. [Task 470][t470]'s pre-claim check must account for this profile; its
port cannot substitute numberless imports or silently disable remote signing.

## Details

- Add optional account XPub and caller-supplied master-fingerprint fields to
  Task 392's `NewAccountParams`, preserving absent versus present-zero
  fingerprint exactly without adding another request or method.
- Accept this shape only for uniformly watch-only SQL wallets and only with an
  explicit exact account number. Spendable wallets and private supplied
  account keys remain unsupported.
- Require a public key with the correct network version, depth three, and a
  hardened child number matching the exact account. Treat the requested
  purpose and coin as caller-declared scope metadata because neither the XPub
  nor optional fingerprint can prove those hardened ancestors.
- Persist the account with `is_derived=false`, its real account number, public
  key, optional fingerprint, scope, schema, and requested `NoChainSync`.
  Introduce no new provenance enum or column.
- Own the two-dialect migration that relaxes the existing account-shape check
  to admit a non-root-derived account with a non-null real account number.
  Preserve the root-derived numbered and arbitrary imported-XPub numberless
  shapes and the existing scope/account-number uniqueness rule.
- Update every maintained AccountStore, AddressStore, and UTXOStore SQL query,
  row-shape check, and public conversion that currently treats
  `is_derived=false` as equivalent to a null account number. Keep address
  derivation-path presence separate from account root provenance, and retain
  the real account number through address and UTXO results.
- Keep SQL source and generated output in their repository-required separate
  commits. Do not edit generated files manually or add a table, column, or
  second persisted discriminator.
- Reuse custom-scope validation, wallet-wide XPub collision admission, cursor
  handling, and NewAccount's conservative `ErrIndeterminateCommit` mapping.
  Return no account on uncertainty; add no readback or automatic retry.
- Preserve ordinary `ImportAccount` as the arbitrary numberless imported-XPub
  operation. Do not reinterpret its rows or fabricate account zero.
- Reuse the existing Signer and AddressManager behavior. Do not add a
  key-family manager, derivation service, lookup service, signer adapter, or
  purpose-1017 storage.
- Return Task 378's canonical `ErrAccountOperationUnsupported` before mutation
  for modern kvdb or a valid supplied-XPub shape that wallet custody cannot
  support. Preserve Task 392's occupied-name precedence before this backend or
  custody check. Map invalid request combinations to `ErrInvalidParam` and
  invalid account keys to the existing `ErrInvalidAccountKey`; add no task-
  specific sentinel.
- Keep implementation local to existing AccountManager, account/address/UTXO
  Store queries and conversions, the required two-dialect migration, generated
  outputs, and focused tests. Private helper and file layout are implementation
  choices, not additional API or storage outcomes.
- Cross-link [Task 464][t464] as the imported-`wallet` account-query owner and
  [Task 463][t463] as the public UTXO-attribution owner for this newly
  representable account shape. Keep focused colocated coverage for path-bound
  allocation and address-result conversion in this production task when
  `NoChainSync=false`: `NewAddress` and `GetAddressInfo` retain the real
  derivation, list lookup retains membership and balance, and reopen preserves
  the same result.
- Adapt Task 395's existing `AllocateNextKey` to this newly representable
  path-bound XPub shape when `NoChainSync=true`. Reuse the account XPub and
  allocation machinery, preserving the supplied path identity and fingerprint
  without requiring wallet-root provenance or creating a watch obligation.
  Extend existing focused allocation tests for this shape on both SQL
  dialects; do not duplicate Task 395's general allocation matrix.

## Suggested fix

Migrate the existing account-shape constraint and conversions once, validate
the supplied depth-three XPub and path facts at the existing account creation
boundary, then reuse the custom account CAS without a Vault secret lookup or
new persisted identity type.

## Verification

Canonical and purpose-1017 watch-only XPub fixtures with both `NoChainSync`
values reopen with exact account identity on SQLite and PostgreSQL. Wrong
network, depth, account child, wallet custody, collision, and numberless misuse
fail before mutation, while purpose and coin remain caller-declared metadata.
Ambiguous results match `ErrIndeterminateCommit` without an account or retry;
ordinary numberless imports remain unchanged. Migration fixtures preserve all
existing account shapes,
while AccountStore lookup, derived-address point and list conversion, and UTXO
ownership results accept the new shape and retain its real account number on
both dialects; the previous mismatched shapes still fail as corruption. Public
rejection tests prove unsupported custody and kvdb cases match
`ErrAccountOperationUnsupported`, invalid request combinations match
`ErrInvalidParam`, and malformed keys match `ErrInvalidAccountKey` with
`errors.Is`. Focused colocated tests prove `NewAddress` and `GetAddressInfo`
retain the real derivation through reopen and `ListAddresses` retains only
membership and balance on both SQL dialects for `NoChainSync=false`, without
adding a public field. For `NoChainSync=true`, focused `AllocateNextKey`
cases prove fresh child allocation from the supplied XPub, complete truthful
key metadata through reopen, and no chain-watch side effects. Account
identity and reopen fixtures still cover both `NoChainSync` values. Task 463
separately proves public `ListUnspent` and `GetUtxo` attribution.

## Definition of done

Uniformly watch-only SQL wallets can install a truthful exact path-bound XPub
account through the generic account API, with chain synchronization selected
independently. The maintained SQL read and allocation paths accept its
non-root-derived numbered identity, including `AllocateNextKey` results for
`NoChainSync=true`, without a new provenance field, component, table, column,
or kvdb path.

[t332]: ../archive/tasks/332-watch-only-script-policy.md
[t337]: ../archive/tasks/337-imported-xpub-derivation.md
[t378]: ../archive/tasks/378-account-manager-error-boundary.md
[t387]: ../archive/tasks/387-account-creation-identity-adr.md
[t388]: ../archive/tasks/388-runtime-account-xpub-collisions.md
[t390]: ../archive/tasks/390-custom-scope-account-creation.md
[t392]: ../archive/tasks/392-discovery-mode-account-creation.md
[t395]: ../archive/tasks/395-key-only-locator-allocation.md
[t463]: 463-path-bound-utxo-identity-itests.md
[t464]: 464-no-chain-sync-account-query-itests.md
[t470]: 470-lnd-wallet-port.md
