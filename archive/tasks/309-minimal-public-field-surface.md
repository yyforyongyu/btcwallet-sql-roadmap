# Task 309 — shrink the public field surface on the store/wallet types

Status: **NON-EXECUTABLE — field evidence only, superseded by [[Task 313]].**
Its sequencing conflicts with 313's selected field scope; 313's V3/V5 and
"Field surface — introduced fields only" govern. Candidate 3 is in scope,
5 is corrected, 2 is superseded, 1 and 4 are deferred.

Originally recorded from a review pass
over `#1272`/`#1291`/`#1292` after those merged-ready heads went green; every
count and line reference below was verified at `wallet-default-sqlite`
`1243c274c`.

Owner directive that motivates this task: **we want the minimum number of
fields possible.** Several fields on `wallet/internal/db` types and on the
wallet-facing types exist only to disambiguate a value that a better-typed
field would carry on its own. Two of them are also correctness hazards, not
just surface bloat, and those are the reason this is written down rather than
left as taste.

Nothing here is required for the SQLite itest work. It is deliberately
sequenced *after* it.

**Root cause note.** Candidates 5 and 6, and part of the passphrase API shape,
all trace back to one thing: kvdb encrypts under two passphrase-derived keys
while the SQL store uses one. That difference is now settled by owner decision —
**single encryption for every backend** — recorded in [[Task 311]]. The evidence
gathered here is kept under "The dual-key root cause" below, but the conclusion
lives in 311, and candidate 6 becomes a deletion rather than a keep once 311
lands.

## Ranked candidates

### 1. `IsWatchOnly` copies on the `db` types — ADR-sanctioned removal

`data_types.go:462` (`AccountInfo`) and `:837` (`AddressInfo`) carry
`IsWatchOnly bool` that duplicates a wallet-level constant.

ADR 0012 §3 already blesses the removal, in as many words:

> `db.AccountInfo.IsWatchOnly` and `db.AddressInfo.IsWatchOnly` are retained as
> wallet-level convenience copies. They are documented as identical across every
> account or address belonging to the same wallet. **A future cleanup task may
> remove them**; callers that want the canonical reading use
> `Wallet.IsWatchOnly()`.

This is the cheapest win because there is no design question left to settle —
only mechanical caller migration to `Wallet.IsWatchOnly()`, which is cached on
the `Wallet` struct at startup.

- Measured: 81 total references to `.IsWatchOnly`, **23 in production code**.
- **Scope carefully.** Do *not* touch `WalletInfo.IsWatchOnly`
  (`data_types.go:237`) or `CreateWalletParams.IsWatchOnly` (`:306`): the first
  is the canonical wallet-level reading, the second is the create-time input
  that sets it. Only the per-account and per-address copies are duplication.
- Also leave the SQL projection alone. The generated rows select
  `wallets.is_watch_only` as `WalletIsWatchOnly` for other purposes (see
  candidate 6), so removing the Go field is not the same as removing the column
  from the queries.

### 2. `AccountSecret.AccountNumber` is a non-pointer with a fake zero

`data_types.go:509`:

```go
	// AccountNumber is the BIP44 account index for derived accounts.
	// Imported accounts have no account number and leave this field at
	// zero.
	AccountNumber uint32
```

Every other `AccountNumber` on these types is already `*uint32`
(`data_types.go:423`, `:633`, `:669`, `:745`, `:775`). This one is the
exception, and "leave this field at zero" is precisely the fake identity
ADR 0013 exists to remove:

> Imported xpub accounts clear `is_derived` and leave `account_number` NULL
> because they do not have wallet-derived BIP44 identity.

and, in its consequences:

> Imported account code must not collapse SQL NULL to Go zero.

Account `0` is the default derived account, so a zero here is
indistinguishable from a real default-account number. Make it `*uint32` for
consistency with its siblings and with the ADR. This *removes* a hazard without
adding a field.

Worth checking while there whether any signer path reads it for a genuinely
imported account; if none does, the field may be droppable outright rather than
retyped.

### 3. `wallet.AddressDerivation.HasAccountNumber` → `Account *uint32`

`address_manager.go:146`. This is the owner's own suggestion and it is
straightforwardly a field fewer. The `db` layer one level down already models
the same fact as `*uint32`, so today the two layers disagree in style and a
hand-written conversion bridges them (`address_manager.go:360-364`):

```go
	if storeAddr.AccountNumber != nil {
		derivation.Account = *storeAddr.AccountNumber
		derivation.HasAccountNumber = true
	}
```

- Measured: 12 total references, 8 in production, **2 real gates** —
  `address_manager.go:1158` and `psbt_manager.go:2304`.
- Converting deletes the field *and* that conversion block.
- **Keep the reason the bool was introduced.** `info.Derivation` is itself a
  pointer and is nil for raw imports, so the trap being guarded is a caller
  gating on `Derivation != nil` and then emitting a fabricated public account
  `0` for an imported-xpub child. The field doc says exactly that:

  > Callers that emit a public BIP32 path must gate on this rather than on
  > `Derivation != nil`.

  With `Account *uint32` that becomes two levels of optionality and the two
  gates become nil-check-then-deref. Preserve the warning in the doc comment on
  `Account`, and keep both call sites rejecting before any deref.

### 4. `db.AddressInfo.HasDerivationPath` → `Branch`/`Index` as `*uint32`

`data_types.go:802`. Same shape of argument as candidate 3: three fields
(`HasDerivationPath`, `Branch`, `Index`) become two pointers. Net one field
fewer, and `(0, 0)` stops being ambiguous by construction rather than by a
sibling bool.

- Measured: 28 total references.
- Producers differ per backend and both must keep working: SQL sets it from
  `row.IsDerived` (`addressstore_common.go:480`), kvdb from
  `derivation.hasPath` (`kvdb/addressstore.go:1197`), and the derived-address
  create path hardcodes `true` (`addressstore_newderived.go:219`).
- Watch `address_manager.go:315`:
  `storeAddr.HasDerivationPath && storeAddr.Branch == 1` becomes a deref, so it
  needs the nil check kept explicitly.

### 5. `db.AddressInfo.HasScript` — needs a schema-level fix, not a deletion

`data_types.go:830`. **Do not simply remove this one; it is load-bearing on
SQL today.** `db.AddressType` is lossy: `addresstype.FromWallet`
(`addresstype.go:49-52`) maps both `waddrmgr.TaprootPubKey` and
`waddrmgr.TaprootScript` onto `db.TaprootPubKey`, so `AddrType` alone cannot
round-trip back to the wallet type. `HasScript` is the second bit that inverts
the mapping, and `walletAddressTypeMatches` (`address_manager.go:795`) compares
both fields to do it.

The genuinely minimal design is to make the enum injective — add a distinct
`TaprootScript` value to `db.AddressType` — after which `Type` alone
round-trips and **two** fields disappear: this one and
`addresstype.StoreType.HasScript`.

- Cost is why it is ranked last: `AddressType` values are persisted as
  `type_id`, so this is a migration plus backfill, and there are 67 production
  references to `HasScript`.
- Record the fragility regardless of when this is done: `HasScript` conflates
  "is a script-type address" with "has stored script bytes", because SQL derives
  it from a `LEFT JOIN address_secrets` on `encrypted_script` presence (there is
  no `has_script` column). Today those coincide — a watch-only P2TR script
  import still persists an `encrypted_script` under `CKTPublic`, asserted at
  `signer_test.go:1619`. If any future import path ever records a script-path
  address without the blob, that row silently round-trips back as key-path
  `TaprootPubKey` instead of erroring.

### 6. `db.AddressSecret.ScriptIsSecret` — delete once Task 311 lands

`data_types.go:889`. On a store-backed (SQL) wallet this field is inert twice
over:

1. It carries no information — the SQL adapter computes it as
   `!row.WalletIsWatchOnly` (`sqlite/addressstore_getsecret.go:98`,
   `sqlite/addressstore_common.go:35`), a pure function of a wallet-level
   constant. There is no `script_is_secret` column.
2. It cannot change behaviour — `selectUnlockedCryptoKey`
   (`keyvault/vault_cipher.go:79-92`) returns `cryptoKeyScript` for **both**
   `CKTScript` and `CKTPublic`, because a store-backed wallet has no separate
   always-available public key. `signer_test.go:1610` says so directly: "the
   round trip below succeeds either way".

It is load-bearing **only** on dual-encryption kvdb wallets. There `CKTPublic`
and `CKTScript` are different keys with different lock-time availability
(`waddrmgr` zeroes `cryptoKeyScript` on `lock()` but deliberately keeps
`cryptoKeyPub`, `manager.go:477-484`), the flag is persisted per address, and
`ImportTaprootScriptDeprecated` takes `isSecretScript` from its caller while
`waddrmgr` rejects only the secret direction — so one spendable kvdb wallet can
hold scripts of both kinds.

[[Task 311]] removes that reason: under single encryption there is one key, so
the flag has nothing to select. **Sequence this after 311**, then delete the
field from the shared contract along with `scriptCryptoKeyType`
(`signer.go:835`).

**Until then, the migration caveat stands.** No code moves script ciphertext
today (verified: no kvdb→SQL secret migration exists). Whichever conversion
runs — the thin in-kvdb dual→single migration or the eventual kvdb→SQL one —
must **re-encrypt** each blob under the destination's single key. Copying
ciphertext verbatim leaves blobs sealed under `waddrmgr`'s public key while
`!WalletIsWatchOnly` reports `true` for a spendable wallet, so the read side
decrypts under the wrong key and fails the ciphertext's authentication tag.

## The dual-key root cause

**Conclusion superseded by [[Task 311]]: single encryption is the invariant for
every backend, and new kvdb wallets use it too.** The findings below are kept
because 311's implementation depends on them; the earlier recommendation
against convergence no longer applies.

**What the two keys buy kvdb today.** The difference is not only *how many
passphrases* but *what is encrypted at rest and when it is readable*:

- kvdb encrypts **public** material under `cryptoKeyPub`: account extended
  public keys (`scoped_manager_accounts.go:58`, `:197`;
  `scoped_manager.go:2102`, `:2206`, `:2430`) and address hashes / script
  identities (`scoped_manager.go:1488`, `:1738`, `:2713`), read back at `:807`,
  `:1100`, `:1133`, `:1148`.
- The SQL store keeps the same material **plaintext** — `MasterPubKey`
  (`data_types.go:253`), account `PublicKey` (`:477`, `:514`), `ScriptPubKey`
  and `PubKey` (`:820`, `:823`). It therefore needs no public key at all.
- `waddrmgr` deliberately keeps `cryptoKeyPub` live across `lock()`
  (`manager.go:484`: "intentionally not cleared"), which is what lets a
  **locked** kvdb wallet still serve addresses and xpubs. The SQL vault has no
  equivalent and refuses while locked.

So "one key" is really "public material is not encrypted", which is why 311 has
to decide explicitly whether kvdb's public material becomes plaintext at rest
(preserving locked reads) or is sealed under the single key (losing them). That
open question is tracked in 311, not here.

**Migration timing.** `Open(ns, pubPassphrase, chainParams)`
(`manager.go:2277`) takes only the public passphrase — a wallet is opened
*locked*. Re-encrypting anything under the private or script key requires the
private passphrase, so the conversion must run at **first unlock** and be
crash-resumable. 311 records this.

**lnd is a second party to the format.** lnd co-writes btcwallet's `waddrmgr`
bucket through a hardcoded namespace key, so any on-disk change to that bucket
is a coordinated change with lnd, not a btcwallet-local one.

## Not in scope

- `wallet/deprecated.go`, `wallet/interface.go`'s aggregate `Interface`, `rpc/`
  and `legacyrpc` — out of scope by standing directive. See [[Task 310]] for
  removing this round's additions to the deprecated files.
- `WalletInfo.IsWatchOnly` and `CreateWalletParams.IsWatchOnly`, per candidate 1.
- Anything requiring a new numbered migration on top of the unmerged
  feature-branch schema; ADR 0013's implementation notes say to edit the
  existing migrations in place, which candidate 5 must follow.

## Suggested sequencing

1 and 2 are independent, mechanical, and safe to land together. 3 and 4 are one
coherent "optional means pointer" pass over the derivation metadata. 5 is schema
work and should be sized on its own. 6 is now blocked on [[Task 311]] and
becomes a straight deletion once it lands.

## References

- [ADR 0011](https://github.com/btcsuite/btcwallet/blob/4236325cbb609881272f488e413b90ab4aa07fc6/docs/developer/adr/0011-no-addresses-used-column.md):
  compute, don't persist — the pattern candidates 4 and 5 already follow.
- [ADR 0012](https://github.com/btcsuite/btcwallet/blob/4236325cbb609881272f488e413b90ab4aa07fc6/docs/developer/adr/0012-wallet-level-watch-only-uniformity.md):
  wallet-level watch-only; explicitly permits candidate 1. Its kvdb
  grandfathering is narrowed by [[Task 311]] to pre-existing wallets only.
- [ADR 0013](https://github.com/btcsuite/btcwallet/blob/4236325cbb609881272f488e413b90ab4aa07fc6/docs/developer/adr/0013-normalized-account-address-identity.md):
  normalized identity; "imported account code must not collapse SQL NULL to Go
  zero" is candidate 2.
