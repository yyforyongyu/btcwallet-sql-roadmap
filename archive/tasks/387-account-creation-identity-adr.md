# Task 387 — define account identity and chain-sync policy

Component: **ADR 0012 and ADR 0013 account-creation contract**.

Type: **Design**.

Dependencies: **[Task 332][t332] and [Task 360][t360]**.

## Rationale

Custom root-derived accounts and exact external XPub accounts need truthful
identity, while purpose-1017 accounts need to opt out of chain synchronization
without inventing a new signing or custody model.

## Details

- Amend ADR 0013 to distinguish root-derived numbered accounts, arbitrary
  numberless imported-XPub accounts, and exact path-bound external-XPub
  accounts whose known hardened account component is real but whose custody is
  not root-derived.
- Preserve `is_derived` as one provenance fact: true means the account XPub was
  derived from the wallet root. Represent a path-bound external depth-three
  XPub with `is_derived=false` and its real non-null account number; add no
  provenance enum or column.
- Define `NoChainSync` as an immutable boolean orthogonal to wallet-level
  signing custody. Both spendable and uniformly watch-only wallets may have
  accounts with either value; false includes the account in automatic chain
  synchronization and true excludes it.
- Define `NoChainSync` narrowly: it excludes the account's derived scripts from
  automatic discovery and live script registration. It does not change
  address ownership, public derivation, or signing capability, and it does not
  discard an owned transaction learned through an existing ingestion path.
  The wallet signs whenever its existing custody model supplies the required
  private operation.
- For a depth-three external account XPub, validate its network, depth, and
  hardened child against the requested account number. Treat purpose and coin
  as caller-declared scope metadata because neither the XPub nor its optional
  fingerprint can prove those hardened ancestors.
- Amend ADR 0012 only as needed to retain uniform wallet-level custody: a
  supplied XPub account is accepted in a uniformly watch-only SQL wallet and
  is not a route for public-only material into a spendable wallet.
- Define one extensible `NewAccountParams` operation. [Task 392][t392]
  introduces it with the existing scope and name inputs plus `NoChainSync` and
  adds `NoChainSync` to `AccountInfo`; Tasks 343, 390, and 391 later add and
  own the optional exact-number, schema, and supplied-XPub/fingerprint fields.
  Do not publish dormant fields, add an lnd-specific API, or add a key-family
  manager.
- Preserve the existing ordinary numberless-import boundary as
  `ImportAccount(context.Context, string, *hdkeychain.ExtendedKey, uint32,
  waddrmgr.AddressType, bool) (*AccountInfo, error)`. Its fingerprint is a
  required value, including present zero; the XPub version and address type
  retain their current scope and schema selection. It has no account number,
  schema override, or `NoChainSync` input and always creates
  `NoChainSync=false`. Do not add an `ImportAccountParams` wrapper for behavior
  that the operation does not support.
- Keep the future `IsWatchOnly` to `CanSign` terminology change out of scope;
  it is tracked by [roadmap issue 22][issue22].
- Amend only the existing ADR files. Do not create another ADR or prescribe
  new components, interfaces, helpers, tables, or test files.

## Suggested fix

Record one focused amendment across ADR 0012 and ADR 0013, including an
identity table and the four wallet-custody/`NoChainSync` combinations, while
reusing the existing account, address, and signer ownership boundaries.

## Verification

Walk generated-seed, imported-root, watch-only depth-three XPub, arbitrary
numberless XPub, canonical-scope, and custom-scope examples through the
identity contract. Confirm all four signing-custody/chain-sync combinations
are coherent, Task 332 remains intact, and no new implementation component or
persisted identity fact beyond `NoChainSync` is required.

## Definition of done

Accepted ADR text fixes the final account identity, new-account request,
retained import signature, result, and chain-sync contracts so implementation
tasks need no product decision, while changing no Go or SQL source and adding
no new ADR file.

[issue22]: https://github.com/yyforyongyu/btcwallet-sql-roadmap/issues/22
[t332]: ../archive/tasks/332-watch-only-script-policy.md
[t360]: ../archive/tasks/360-document-kvdb-adapter-identity.md
[t392]: 392-discovery-mode-account-creation.md
