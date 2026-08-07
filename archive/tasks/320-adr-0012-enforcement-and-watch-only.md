# Task 320 — enforce ADR 0012 and fix watch-only creation paths

**Status:** deferred, not scheduled. Split out of [[archive/315-rebuild-minimal-dual-backend-itest-stack.md]] (archived) §4.

## Why deferred

Both are import/creation-policy questions. The Manager itest creates a
generated-seed spendable wallet and never imports anything, so neither is
reachable on the milestone path.

## ADR 0012 — a script is not signing material

ADR 0012 says script presence never substitutes for a private key. Canonical
kvdb does not enforce it: an address import that supplies only a script can
still be admitted as spendable, and the wallet then fails at signing time rather
than at import time.

Enforce it at the import boundary in canonical kvdb, matching what canonical SQL
already does. Tighten `addressstore_newimported` to a true private-key
requirement.

**Do not** reuse `hasScriptKey` to answer this question. The previous attempt
did, and the result was that a watch-only kvdb wallet could no longer import
**any** script — contradicting the ADR it was implementing. The two questions
are "can this wallet hold script bodies" and "does this address have signing
material", and they need separate predicates.

### The gap this leaves once SQLite is the default (verified 2026-07-31)

`RequireAddressPrivKeyOnSpendable` counts only `EncryptedPrivateKey`, so on a
**spendable SQL wallet** every script-only import is refused. Verified against a
real SQLite Manager wallet:

```
w.ImportTaprootScript(...) ->
  spendable wallet 1 cannot import address without private-key material into
  account "imported": spendable wallet must not contain an imported address
  without private-key material
```

This is **base behaviour**, not something task 315 introduced — the predicate is
byte-identical at `sql-wallet` `2ea76dfc5`; only the surrounding comments were
rewritten. kvdb does not apply the check, so the same exported method succeeds
there (task 315 added
`TestImportTaprootScriptKVDBReadableWhileLocked` proving the kvdb round trip).

It matters more now that #1292 makes SQLite the default, because taproot script
import is lnd's primary use of the method. Decide it here, at the wallet layer,
so both backends agree:

- if a spendable wallet genuinely may not hold script-only imports,
  `Wallet.ImportTaprootScript` and its witness-script sibling should refuse up
  front with a documented API-level error, instead of surfacing a store-shaped
  one;
- if they are meant to keep working, the script-only case needs an explicit
  waiver — a retained script is a watched spending *condition*, and the wallet
  refusing to sign it later is the correct outcome, not refusing to track it.

Note `wallet/address_manager_test.go`'s `TestImportTaprootScript` runs against a
mock store that returns success, so it cannot catch either shape.

## Watch-only creation modes

Two validator gaps, both pre-existing:

- a create request can pass a **private** root key with watch-only mode set. The
  layer below then builds the wrong manager. Reject the combination at
  validation rather than supporting it;
- a watch-only xpub creation mode is admitted that the layer below mishandles.
  Same disposition: reject the mode, do not broaden the layer below to
  accommodate it.

315 deliberately drops the previous attempt's watch-only broadening for this
reason — rejecting an unsupported mode is smaller and safer than making it work.

## Also here: the typed Vault cipher redesign

315 §7 Q4 keeps base's `Encrypt`/`Decrypt(CryptoKeyType, ...)` for the milestone,
because the typed private/script redesign measures 542 `A+D` across 12 files and
is a boundary cleanup rather than a prerequisite for removing walletdb
transactions. It belongs with this task's public script-import cleanup, since
both are about giving script handling an honest API. `db.AddressSecret.ScriptIsSecret`
stays regardless — it names which key encrypted a given ciphertext, which the
signer needs either way.

## Definition of done

1. A script-only import is refused at import time on canonical kvdb, with the
   error naming ADR 0012.
2. A watch-only kvdb wallet can still import scripts — the specific regression
   the previous attempt introduced, tested explicitly.
3. Watch-only + private root, and the mismatched xpub mode, are both rejected at
   validation with distinct errors.
4. Every case falsified.

## Evidence

`docs/developer/adr/0012-*.md`; `archive/313-single-encryption-revision-of-the-three-prs.md`
V1 (the reversal: enforce rather than amend the ADR). The regression is
described in the #1272 thread on `wallet/internal/db/kvdb/addressstore.go:847`
in `.lgtm/pivot/t314-thread-dispositions.md`.

## Carried forward from the delivered stack (codex, 2026-08-02)

**Private-root `WatchOnly` requests differ by backend.** For `WatchOnly=true`
with a generated seed, imported seed, or XPrv, kvdb passes the private root
through while SQL does not, so the same request produces different
effective-watch-only state per backend. Define one shared rule; the smallest
fix is to normalise the request before it reaches either backend rather than
teaching each backend the same policy.
