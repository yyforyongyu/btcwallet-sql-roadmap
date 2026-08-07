# Task 326 — preserve imported-xpub public derivation

**Status:** deferred, not scheduled. Raised by the post-delivery review of
#1291. It does not block the existing `manager_create_wallet` itest, which does
not import an account or call `DerivePubKey`.

## Problem

The pinned-base `Wallet.DerivePubKey` delegates to waddrmgr
`DeriveFromKeyPath`, whose internal account number can identify an imported-xpub
account. The Store-routed implementation instead calls `Store.GetAccount` with
`GetAccountQuery.AccountNumber`.

The shared query correctly rejects an imported row selected by public BIP44
account number: ADR 0013 requires imported-xpub accounts to have no public
`AccountNumber`. The result is that public child derivation now returns
`ErrAccountNotInStore` for an imported-xpub account even though its account
xpub is stored and sufficient for the operation. This matters to watch-only
and remote-signer consumers such as lnd.

This is a public-key lookup problem, not a private-key exception. Imported-xpub
accounts still have no account-level private material, and private signing must
continue to fail without fabricating account zero.

## Why deferred

The first SQLite itest only creates, starts, syncs, and stops a generated-seed
wallet. Fixing this correctly requires an explicit imported-account identity
contract at the public signer boundary; adding another Store selector to the
three current PRs would expand their API and test surface.

## What to do

1. Define how a public derivation request identifies an imported account on
   both backends. Do not silently reinterpret `BIP32Path.InternalAccount`:
   kvdb `AccountID` is a waddrmgr internal number, while SQL `AccountID` is a
   relational row ID.
2. Choose the smallest explicit API shape, such as a separate account name or
   durable account identity on a public-derivation request. Keep ordinary
   derived-account calls source-compatible if possible.
3. Resolve the account xpub through `db.Store` and derive branch/index in
   memory. Do not restore `walletdb.View`, route through `deprecated.go`, or
   require a legacy scoped manager.
4. Keep the normalized identity rules from ADR 0013: imported-xpub
   `AccountNumber` remains nil, and SQL row identity never becomes a BIP44
   number.

## Tests

- SQLite imported-xpub public child derivation succeeds while locked.
- kvdb imported-xpub public child derivation preserves pinned-base behavior.
- A derived account still resolves by BIP44 account number.
- A collision between a derived account number and an imported Store identity
  cannot select the wrong xpub.
- Private derivation/signing for the imported-xpub account still returns the
  existing no-private-key error.

Add focused unit or backend tests only. A new daemon itest is not part of this
task unless a later integration milestone explicitly requires one.

## Definition of done

1. `DerivePubKey` works for imported-xpub accounts on kvdb and SQLite without
   `walletdb.Update/View` in the new wallet components.
2. No new ambiguous presence booleans or fabricated account numbers are added.
3. ADR 0013 identity semantics remain intact.
4. The derived-account and private-key paths do not regress.

## Evidence

Post-delivery #1291 review thread on `wallet/signer.go`:
<https://github.com/btcsuite/btcwallet/pull/1291#discussion_r3698833138>.
Task 1000 is reference material only; this file is the focused deferred owner.
