# Task 313 — revise the three PRs for the single-encryption directive

> **SUPERSEDED by [[Task 314]] for anything not yet done.** The
> single-encryption arc in this file is delivered and pushed; its execution log
> is the record of that. What remained — transactional create, the lifecycle
> work, the review followups — is governed by 314, whose objective (minimise the
> **total** diff from `sql-wallet`) and whose owner decision to use one database
> transaction *instead of* locks and intermediate states both change the shape of
> R4/R8/R9 recorded here. Treat this file as evidence.

Status: **DELIVERED (single-encryption arc) / SUPERSEDED (the rest).**
Originally: READY TO EXECUTE. Every decision is closed:

- **kvdb format option (a)** — approved explicitly (owner, 2026-07-30). Canonical
  kvdb gets a distinct credential-aware waddrmgr single-encryption create/open
  format at the **reserved next namespace `mgrver`**; the legacy migration list
  and legacy create/open are untouched, and the generic credential-free migration
  path never advances a wallet to it.
- Manager-owned Store, `NewManager` retained; one Manager assumed.
- `Wallet.Stop` restartable; `Manager.Close` runs no wallet shutdown.
- kvdb single-wallet; migration out of scope; deprecated files zero-touch;
  ADR 0012 enforced.

No further architectural review is required — all three reviewers concur.

Tasks 305–312 are historical evidence and finding inventories only. An
instruction from them executes **only when this file explicitly says to carry it
forward or implement it** — citations, evidence, and moot/reversed dispositions
never reactivate an old instruction. Task 308 is archived in full, including its
"Revision 2". See [[Task 300-INDEX]].

Source: three independent branch reviews (`gpt-review-task.md`,
`fable-review-task.md`, `codex-review-task.md`) against
`impl-secret-store` `0d98a7d45`, `impl-runtime-store-6` `8b4f29f74`,
`wallet-default-sqlite` `1243c274c` — CI-green on all three exact heads.

**Review-thread state (corrected — gpt finding 11).** An earlier revision of
this file said "zero unresolved threads". That was stale. Current state is
**#1272: 6, #1291: 2, #1292: 0** — the eight triaged in [[Task 312]]. #1291's
two are on **page 2** of `reviewThreads`, invisible to a single `first:100`
query. #1272's PR body still carries the two inaccuracies V4 records.

**The existing branches are incomplete, and all three reviewers agree.** They are
internally consistent and green; the verdict comes from [[Task 311]]'s directive
arriving *after* they were built. This supersedes my earlier "305 and 308 are
finished" report: that work was finished against the previous premise.

## Execution: rebuild, don't unwind

Owner directive: **we can always recreate everything from scratch.** Treat the
current commits as a reference, not a constraint.

- Deletions become "never write it" rather than "unwind it".
- The commit sequence gets designed for reviewability from the start.
- It eliminates the modify-then-delete churn codex flagged (`U1UGB`) by
  construction.
- No cherry-pick conflict archaeology.

**Safety net.** The current green heads are pinned:

```text
refs/backup/t313-green/impl-secret-store      0d98a7d45
refs/backup/t313-green/impl-runtime-store-6   8b4f29f74
refs/backup/t313-green/wallet-default-sqlite  1243c274c
```

(verified equal to `upstream` at creation; pre-rewrite originals at
`refs/backup/t308/*`).

**Verification method:** diff the rebuilt tree against
`refs/backup/t313-green/*` and account for **every** difference as either
intentional or a carried-forward fix. Anything unexplained is a regression. The
risk of starting over is not the code — it is silently losing fixes that are
still valid, which is why the inventory below exists.

## Carry-forward inventory

### Must survive the rebuild

| Item | Why it still applies |
| --- | --- |
| **F1** accepted-request completion — `waitForAcceptedResp`, cancellation-boundary docs, the table test, `TestAcceptedRespInvariants` | The Controller request protocol is independent of encryption |
| **Unbuffered `requestChan` in the test helper** (`common_test.go`) | Base-level fidelity fix; F1's invariant test is meaningless without it |
| **`UfRny`** register-before-`Start` in `bwtest/harness_wallet.go` and `itest/manager_test.go` | Harness correctness |
| **`UfRns`** dead `scanTarget.AccountName` removed, assertions retargeted at `info.AccountName` | Preserves the "which account resolved" coverage |
| **`UfSB5`** `InitialAccounts` doc correction | Base-level doc fix |
| **F5** `testRootKey` per-test master key | The hazard is five *other* pre-existing parallel tests sharing a mutable key — independent of the rotation tests that die |
| **F6** conversion tests moved into `db: add account secret types` | About account-secret types, not encryption |
| **`ErrInvalidPassphrase`** exported on `wallet` | Single-passphrase model still needs a matchable sentinel |
| **`wallet: re-export TxStatus for external callers`** | External callers cannot import `wallet/internal/db` |
| **#1292's two commits** | Scope complete; replay onto corrected heads |
| **`RescanTarget`** stable-identity change | Keep — as **its own commit** (§D) |
| **Process discipline** — per-commit `go vet`, four pinned lint passes, 80-col at tab-4, `fmt-check`/`sqlc-check`, GPG signing | Independent of content |

### Dies deliberately

- **F0** — the two-passphrase `ChangePassphraseParams`, the public half of
  `ChangePassphraseRequest`, `LegacyManagerVault`'s dual delegation, and the two
  tests restored for it.
- **T7** — `atomicPassphraseRotator`, `errAtomicRotationUnsupported`, its
  negative test. Retires most of [[Task 310]].
- **`waddrmgr.ChangePassphrases`** staged atomic rotation and its two
  concurrency tests — which moots [[Task 312]] §A, the incomplete locked-state
  wipe. Deleting the code beats fixing the bug in it.
- **`ScriptIsSecret`** plumbing through `db.AddressSecret`, `address_manager.go`
  and `signer.go`; **`scriptCryptoKeyType`**.
- **`wallet.ErrPublicPassphraseUnsupported`** and `keyvault`'s equivalent.
- **C4's** `isKVDB` comment — backend-inferred vault selection goes away
  entirely. The kvdb opener checks its namespace `mgrver` as an **admission**
  test and refuses legacy versions; no `LegacyManagerVault` is constructed.
- The **410-line Start/Stop teardown protocol** and the **partial-create replay**
  machinery.
- **Never write** `(*kvdb.Store).DB()` (312 H2 — dead on arrival) or
  `kvdb.IsMissingStore` (312 H1 — map `walletdb.ErrDbDoesNotExist` onto a
  `wallet/internal/db` sentinel from the start, preserving the walletdb import
  boundary).

### Must be re-decided, not carried

- **V1** — enforce ADR 0012 (conclusion reversed below).
- **V2** — fix the repeat-unlock regression properly.
- **Which code TODOs remain.** `Manager.Load` serialization, per-wallet pools and
  non-atomic creation are *fixed* here rather than documented; the
  `db.AccountInfo` leak and `Rescan` scoping remain as TODOs.

## Verified findings

### V1. ADR 0012 contradiction — script-only imports (blocker)

**Confirmed.** `RequireAddressPrivKeyOnSpendable`
(`wallet/internal/db/addressstore_newimported.go`) accepts a script-only import
into a spendable wallet:

```go
	scriptSpendable := hasScript && addrType.IsScriptSpend()
	if walletIsWatchOnly || hasPrivKey || scriptSpendable {
		return nil
	}
```

ADR 0012:96-98 rejects exactly that: "A public-only **or script-only** address
import is rejected with `ErrSpendableWalletNeedsAddressPrivKey`." The function's
own doc claims to enforce the invariant while inverting it. **Introduced in this
stack** — `scriptSpendable` has zero hits at `sql-wallet`.

**Recommendation reversed: enforce the ADR, do not amend it.** I previously
recommended amending 0012 on the grounds that a P2WSH witness script "is the
spend material". gpt and codex both rejected that, and they are right: a redeem
or witness script describes a spending *condition*, not the signing material
needed to satisfy it. A 2-of-3 multisig script is not spendable merely because
the wallet holds its bytes. Script-byte presence is not a spendability model.

So:

1. Require `encrypted_priv_key` for every imported address in a spendable wallet.
2. Remove the P2SH / P2WSH / taproot tests that assert spendable script-only
   imports are accepted.
3. Keep accepting script-only imports in a uniformly watch-only wallet.

Script-only spendability, if wanted, needs its own domain model and a deliberate
ADR update — not a silent waiver inside a store validator.

### V2. Repeat-unlock destroys an unlocked session (regression)

**Confirmed**, `handleUnlockReq` in `wallet/controller.go`:

```go
	// Lock first so Unlock always validates the passphrase: ...
	w.keyVault.Lock()

	err = w.keyVault.Unlock(w.lifetimeCtx, req.req.Passphrase)
```

A repeat `Unlock` with a **wrong** passphrase locks a wallet that was unlocked,
discarding the session and its auto-lock timer. Any caller able to submit a bad
passphrase can lock a running wallet.

Minimal correction: **remove the lock-before-unlock change**, restoring
pre-#1291 behaviour. A non-mutating authentication operation can be designed
later; it should not be improvised inside store plumbing.

### V3. Public `wallet.AddressInfo.HasScript` is write-only

**Confirmed.** `addresstype.ToWallet(db.TaprootPubKey, hasScript=true)` returns
`waddrmgr.TaprootScript`, so the disambiguated answer is already in `AddrType`.
Every production `HasScript` read is of an **internal** type feeding that
conversion — `walletAddressTypeMatches(info db.AddressInfo, …)`
(`address_manager.go:793`) and `db.UtxoInfo` (`utxo_manager.go:234`). The public
field is written at `address_manager.go:323` and never read.

This corrects [[Task 309]] candidate 5, which conflated the public copy with the
internal projection. **The internal projection stays** (it feeds `ToWallet`); the
public field goes.

### V4. #1272's PR body is false on two counts

**Confirmed.** It says "content-preserving, no commit changes. Stacked on
`impl-runtime-store-5`." The rewrite was neither, and `baseRefName` is
`sql-wallet`. A rebuild makes the first claim less true still.

### V5. `db.AccountSecret` can drop five of its seven fields

Raised by gpt (9) and codex (4). **Confirmed by reader count** — production
references: `PublicKey` 2, `EncryptedPrivateKey` 2, and **zero** for `WalletID`,
`Scope`, `AccountNumber`, `AccountName`, `MasterKeyFingerprint`. The query
already carries wallet and account identity.

This **supersedes [[Task 309]] candidate 2**, which proposed retyping
`AccountNumber` to `*uint32`: deleting the field is strictly better than fixing
its optionality. Tests asserting those projection copies get rewritten around the
actual signer contract.

**Six, not five — `PublicKey` goes too. Settled.** The type's own contract says
it carries no plaintext material, yet `PublicKey` is plaintext. The check I had
deferred to rebuild time is now closed:

- `resolveDerivedPubKeyFromStore` has **exactly one caller** (`signer.go:489`),
  on the public `Wallet.DerivePubKey` path, which identifies the account by BIP44
  account number.
- `GetAccount` accepts that same wallet/scope/account-number selector, returns
  `AccountInfo.PublicKey` on SQLite, PostgreSQL **and** kvdb, supports
  `SkipBalance: true` (`data_types.go:646`) for one backend read, and works while
  locked because the xpub is public metadata.
- Imported-xpub accounts are not a counterexample: ADR 0013 gives them no public
  BIP44 number, and `GetAccountWithOps` deliberately rejects number-selected
  imported accounts — `DerivePubKey(BIP32Path)` has no selector that could name
  one.

So the minimum contract is:

```go
type AccountSecret struct {
	EncryptedPrivateKey []byte
}
```

Rebuild work: point `resolveDerivedPubKeyFromStore` at
`GetAccount(..., SkipBalance: true)`, and drop the public-key and identity
columns from the account-secret projections where they are no longer read.

## Field surface — introduced fields only

Per codex, this round trims only what these PRs introduced:

1. Remove public `wallet.AddressInfo.HasScript` (V3).
2. Replace `AddressDerivation.Account` + `HasAccountNumber` with
   `AccountNumber *uint32` ([[Task 309]] candidate 3).
3. Remove `db.AddressSecret.ScriptIsSecret` and every wallet-level crypto-key
   selection branch.
4. Shrink `db.AccountSecret` to **only `EncryptedPrivateKey`** — six fields
   removed (V5).

**Explicitly out of scope:** 309's pre-existing `IsWatchOnly` copies,
`HasDerivationPath`, and the `db.AddressType` enum work. Also **do not** remove
public `Imported`, `Internal` or `Compressed` — gpt finding 7 shows
`AddressInfo.Imported` has production readers (`signer.go:716`,
`address_manager.go:1138`), so that is a behaviour change, not a dead-field
deletion.

## Convergent architectural findings

### A. The directive is not satisfied

`Manager.buildWallet` selects `LegacyManagerVault` from `isKVDB`, so a **new**
kvdb wallet still gets waddrmgr's two-master-key / three-crypto-key hierarchy.
Passing the same bytes as both legacy passphrases would not fix it: the two
salted master keys, `CKTPublic`/`CKTPrivate`/`CKTScript` and per-record key
selection would all survive.

**Corrected mechanism (all three reviewers converge):** reuse waddrmgr's
existing **`mgrver`** as the format discriminator. **No** new format field,
registry, manifest, completion marker or write lock — this replaces the "explicit
encryption-format marker" an earlier revision of this file proposed. Vault
**admission** keys on the persisted kvdb-namespace version, never on `isKVDB`;
an admitted wallet constructs only the canonical vault.

**Legacy-dual wallets are refused this round.** `Manager.Load` returns a stable
migration-required error; the credential-gated converter is a future PR
([[Task 311]]). The single-passphrase public API must not silently open a wallet
whose public and private credentials differ.

### B. Task 305's F0 is reversed

F0's justification was that one API must serve old kvdb wallets during the
transition. 311 removes that premise, so the two-passphrase params, the public
half of the request, and `LegacyManagerVault`'s dual delegation stop being
deliverables. This also settles `U06ol` ([[Task 312]] §G): the flags are
**deleted**, not inferred from empty byte slices.

### C. #1291's complexity compensates for two wrong choices

1. **Every `Wallet` owns opened store handles and closes them from `Stop`** — the
   410-line teardown protocol exists to make that safe.
2. **`Manager.Create` writes across several transactions** — so partial-create
   replay, persisted-root recovery, retry-key comparison and publication guards
   exist to recover from it.

**Atomic creation is unanimous.** One backend transaction inserts wallet
metadata, wallet secrets, initial sync state, key scopes, default and imported
accounts, and account secrets. Validation, derivation and encryption are staged
before the write; Manager never receives a raw transaction; publication happens
only after commit. Then **delete rather than repair**: pre-existing-row
classification, partial-create replay, persisted-root recovery, retry-key
comparison, created counters, create-specific unstarted cleanup. A failed
transaction leaves nothing to recover. Extending `Store.CreateWallet` to take the
prepared aggregate is sufficient; a second public method is unnecessary.

Codex's observation closes the argument for change: the 1,200-line Manager
expansion **still doesn't achieve its own invariant**, because the guard is taken
in `Create` but not `Load` ([[Task 312]] §B).

### C-bis. Store ownership — DECIDED: Manager-owned

Resolved. All three reviewers converged on Manager-owned, and the owner
direction confirms it with the `NewManager` name retained.

**Evidence that settled it:** ADR 0001 commits to "multiple wallets within each
network" (a hot and a cold wallet on one mainnet database), so a pool per wallet
breaks a *supported* use case — which is also known-limitation #2 in the round-1
PR body ("one wallet per file for now"). codex withdrew its per-wallet position
on exactly this ground; fable switched to match.

**Caller-owned Store was considered and rejected**, not merely skipped, on
bounded-scope grounds: `db.Store` and the backend constructors are internal, so
caller ownership needs a new public opaque Store API plus constructors — public
surface deferred rather than forced into these three PRs. It can be added
deliberately later if lnd needs to share an application pool.

(The "a store handle cannot stop the wallets running on it" argument is
**retired** — `Manager.Close` does not stop them either, so it no longer
distinguishes the alternatives. See R8.)

The full contract — constructor, coarse mutex, `Close` sequence, restartable
`Stop`, and what is explicitly *not* added — is in **R8**. It is not restated
here, so there is one authority for it.

### D. Reconciling the apparent `Rescan` disagreement

gpt says the `[]waddrmgr.AccountScope` → `[]RescanTarget` change "should not be
bundled into this migration commit"; codex says it "should stay". Compatible:
**keep it, in its own commit.**

## Status of tasks 305–312

Non-executable. Use as evidence only.

| Task | Status |
| --- | --- |
| 305 | Round-1 record. F0/T7/F9-public/F5's rotation tests **reversed**; F1 and F6 survive as landed. |
| 306, 307 | Retired reference. 306's atomic-creation item is live again via §C. |
| 308 | Round-1 execution record + cascade mechanics. **Its "Revision 2" is superseded** — its constant-public-passphrase kvdb design is not single encryption, its lifecycle addition keeps wallet-owned stores, and its atomic-create addition excludes canonical kvdb. |
| 309 | Field evidence. Candidate 3 promoted; candidate 5 corrected by V3; candidate 2 superseded by V5; candidate 6 becomes deletion; candidates 1/4 and the enum work are out of scope. |
| 310 | Standing rule survives ("no new code in deprecated files"); its relocation work is mooted — the machinery is never written. |
| 311 | Doctrine, incorporated here. |
| 312 | Thread triage. §A **moot** (machinery deleted). §B fixed by §C/C-bis. §C **moot** — 311 deletes `ScriptIsSecret` rather than documenting it. §D **moot** — kvdb Manager admission keys on namespace `mgrver`, not a TODO. §E valid. §G superseded by §B. §H1/§H2 become never-write rules. |

## Execution log

### 2026-07-30 — owner rulings on the two open dispositions

**Ruling 1 — deprecated.go: minimal change is acceptable, no unnecessary ones.**
The recorded "byte-identical at both heads" rule is amended: compile-forced
changes are permitted, gratuitous ones are not. Audited and trimmed to **+9/−9**,
every line forced:

- the `keyvault` import (its only use is gone);
- `w.cfg.DB` → `w.db` ×2 (`Config.DB` is a `DBConfig`, not a `walletdb.DB`);
- the `keyVault:` swap (see ruling 2);
- the one-line kvdb backend pin (an unset backend now defaults to sqlite).

Reverted as unnecessary: a cosmetic `store` → `legacyStore` rename, and a
five-line comment cut to one.

**Ruling 2 — `LegacyManagerVault` stays; R5 is amended.** The owner's direction
is the minimal implementation.

R5 as written ("the modern runtime constructs only the canonical vault") is
inconsistent with H3 option (a), which the same owner approved and which says
"the kvdb vault becomes a thin single-passphrase adapter over this mode". R5 is
amended to the latter: the adapter **is** the kvdb implementation of the
canonical `keyvault.Vault` contract, not a resurrection of the dual-passphrase
API. Evidence it is not that API: it takes `(old, new)` passphrases, exposes no
`CryptoKeyType`, and keeps its crypto-class choice in a private method.

The alternative was measured rather than assumed, and is recorded here because
it is the better long-term design:

- `keyvault.WalletVault` needs `Store.GetWalletSecrets` / `UpdateWalletSecrets`,
  which kvdb returns `notImplemented` for. Mapping them onto waddrmgr rows is
  straightforward — `CreateSingleEncryption` already writes exactly those fields.
- The obstacle is **five call sites**, all in the kvdb address store's import
  path (`addressstore.go:834` plus the three script imports; `ImportPublicKey`
  needs nothing in this format). An earlier claim in this file that "every kvdb
  read path" needs an unlocked waddrmgr was wrong: the store returns encrypted
  blobs and the wallet layer decrypts through the vault.
- Those sites do a pointless round trip — the vault encrypts, the store
  decrypts, waddrmgr re-encrypts. A waddrmgr import entry point taking an
  already-encrypted blob removes it, and with it
  `transportScriptCryptoKeyType`, `storeHasScriptKey` and `scriptKeyReporter`
  — the machinery behind two of the bugs fixed this round.

**Deferred as a follow-up**, not done here: implement the kvdb secrets surface,
add the no-crypto import entry point, and reduce the adapter to
deprecated-loader-only.


### 2026-07-30 — review round: watch-only single-encryption cluster FIXED

Heads: `impl-secret-store` **2bddcc472** / `impl-runtime-store-6` **97548d076**
/ `wallet-default-sqlite` **4f0047d85**, all folded into
`waddrmgr: add a single-encryption format` (the commit that introduced them).

Five real defects, all from treating the legacy three-class crypto model as if
it still applied to the new format. Two were confirmed by reading the code
before fixing; a sixth surfaced only when the new test ran.

1. **Script bodies written in cleartext.** `importScriptAddress` and the script
   read path picked `cryptoKeyPub` for a "non-secret" script. In this format
   that key is the pass-through metadata codec, so the complete script body
   went to disk unencrypted — directly contradicting ADR 0014. Both sites now
   go through `Manager.scriptCryptoKey`, which ignores the legacy secrecy flag
   for single-encryption wallets, plus `scriptNeedsUnlock` for the read gate.
2. **Watch-only wallets with accounts could not unlock.** `Unlock` decrypted
   every cached account's private key unconditionally. The whole
   private-material section is now skipped when there is no private crypto key.
3. **`deriveOnUnlock` segfaulted on the same path** — found by the new test, not
   by review: it derives a private child of a neutered key and dereferences the
   nil result. Covered by the same skip.
4. **`ConvertToWatchingOnly` bricked a canonical wallet.** `deletePrivateKeys`
   removed the master key params and the script key, both of which
   `OpenSingleEncryption` requires, so the conversion committed a wallet neither
   opener could load again. It now keeps them for this format.
5. **Watch-only passphrase could not be rotated.** The `private && WatchOnly()`
   refusal became `private && !hasSecretKeys()`, and the rotation tolerates an
   absent private key.
6. **Watch-only kvdb script import** was downstream of 1 and 2; an end-to-end
   kvdb test now covers it.

Six regression tests added (`waddrmgr/format_test.go`,
`wallet/internal/db/kvdb/vault_test.go`). The pre-existing tests missed all of
this because they only exercised the account-less watch-only path.

`deletePrivateKeys` was decomposed into `deleteScopePrivateKeys` /
`deleteAccountPrivateKeys` / `deleteAddressPrivateKeys` / `strippedAddressRow`
to stay under the complexity limits once its signature changed.

Verified: per-commit `go vet` (68 steps), six tag combinations, full suite,
`-race`, golangci clean at all three heads, all commits signed, #1272
deprecated diff still empty.


### 2026-07-30 — executed and pushed

All three PRs rebuilt on the single-encryption doctrine and pushed to both
remotes (upstream + origin), bottom-up, one branch at a time.

Heads: `impl-secret-store` **bfbe5ab68** (23) -> `impl-runtime-store-6`
**979e9e828** (33) -> `wallet-default-sqlite` **1a5703890** (35).

Landed:

- **V1** ADR 0012 symmetric invariant (`RequireAddressPrivKeyOnSpendable`).
- **V2** repeat-unlock revert, plus the second half nobody had spotted: a
  refused unlock also called `state.toLocked()`, so removing the pre-unlock
  `keyVault.Lock()` alone was not enough. Regression test covers both.
- **V3/V5** field trims; `AccountSecret` reduced to `EncryptedPrivateKey`;
  pubkey derivation reads the public `AccountInfo` projection.
- **V4** #1272's PR body corrected (was "content-preserving", wrong base).
- **R2** one `Passphrase` on `CreateWalletParams`; `Config.PubPassphrase` and
  `OpenStoreParams.PubPassphrase` gone.
- **R3** `keyvault.Vault` typed by material; `CryptoKeyType` off the canonical
  boundary. New bottom commit `keyvault: type the vault cipher by material`,
  because most of the retype lands on base-owned code.
- **R7** ADR 0014 + index entry.
- **H3(a)** waddrmgr single-encryption format: `CreateSingleEncryption` /
  `OpenSingleEncryption`, reserved `mgrver` 9 outside the migration list,
  plaintext metadata via a codec (twenty existing sites untouched), watch-only
  mirroring SQL, legacy refusal, migration gate in both load paths.
- **Zero-touch** restored: `wallet/deprecated_test.go` byte-identical;
  `deprecated.go` carries only the 4 lines the `Config.DB` retype and the
  sqlite default force. The `Stop`-before-`Start` regression test moved to
  `controller_test.go`.

Found along the way, worth keeping:

- **waddrmgr never populated `cryptoKeyScript` on unlock** — at base *and*
  upstream. Every legacy script ciphertext on disk was written under an
  all-zero placeholder. Not fixable for legacy without making those records
  undecryptable, so only the new format unseals a real script key.
- **`mgrver` 9 collides with `latestVersion+1`**, which
  `TestManagerHigherVersion` used to mean "too new". Retargeted to
  `singleEncryptionVersion+1`.
- **`waddrmgr.AddrStore` is exported**, so the store asks for script-key
  presence through a locally-asserted `scriptKeyReporter` rather than a new
  interface method, matching the `atomicPassphraseRotator` precedent.
- **The two-passphrase atomic rotation primitive is now dead weight.** With one
  passphrase, `LegacyManagerVault` needs only the single-passphrase
  `ChangePassphrase`, so `waddrmgr.Manager.ChangePassphrases` (496 lines) and
  `keyvault.ChangePassphraseParams`'s public half can go. **Not done this
  round** — noted as the first follow-up.

### Follow-up pass (same day) — rotation + never-write items DONE

- **Dead two-passphrase rotation removed.** `waddrmgr.Manager.ChangePassphrases`
  and its staging helpers are gone and the commit that added them is **dropped**
  from the stack (its content became empty, so keeping it would be
  introduce-then-remove). `keyvault.ChangePassphraseParams` collapsed to
  `ChangePassphrase(ctx, oldPassphrase, newPassphrase)`;
  `wallet.ChangePassphraseRequest` to `{OldPassphrase, NewPassphrase}`.
  `ErrPublicPassphraseUnsupported`, `errReadRandSalt`, `masterKeyParams`,
  `DBPutPassphrase` and ten `TestChangePassphrases*` tests deleted with it. Net
  about -1400 lines.
- **Both never-write items fixed**, at the commit that introduced them
  (`kvdb: add runtime store lifecycle adapters`): `(*kvdb.Store).DB()` deleted —
  it had no callers, exactly as 312 H2 predicted — and `kvdb.IsMissingStore`
  replaced by `db.ErrStoreNotFound`, which kvdb now maps
  `walletdb.ErrDbDoesNotExist` onto at its own boundary, with a regression test
  asserting the walletdb error does not leak past it.

**STILL NOT DONE — R4 + R8 + R9 are one indivisible unit.** They are the
lifecycle reshape, not the encryption doctrine, and the DoD couples them: R9's
"a kvdb-backed Manager admits exactly one wallet identity" presupposes R8's
Manager-owned Store, and R8's "NewManager opens the container, Create performs
the aggregate transaction" presupposes R4's real `kvdb.Store.CreateWallet`
(still `notImplemented`). Doing any one alone leaves kvdb in a half-state.

**R8, R9 and R4's container half are BUILT — production code complete and
compiling — on the local branch `t313/r8-wip` (f345c38ed, not pushed, not for
merge).**

Done there:

- `ManagerConfig`, `NewManager(ctx, cfg)` owning one Store, synchronous
  idempotent `Manager.Close()` with the four-step protocol and terminal fence,
  coarse mutex replacing the RWMutex.
- `kvdb.OpenContainer` / `StoreHandle.BindWallet`: the walletdb file opens
  without a wallet in it, so opening the backend is not creating a wallet.
  `LoadLegacyWallet` reports an empty container as `db.ErrWalletNotFound`.
  This is what unblocked R8 — the earlier attempt died because
  `kvdb.OpenStore` required an initialised waddrmgr namespace.
- Restartable `Wallet.Stop`: `onStopped`, the per-wallet store close and the
  `stopped` terminal flag are gone; `ErrWalletStopped` became
  `ErrWalletTerminal` and now means only that the Manager was closed.
- **R9**: kvdb admits one wallet identity per database, in memory, in
  `beginCreate`.
- Every transient store open converted onto the owned handles:
  `createLegacyWallet`, `readExistingLegacyWallet`, `recoverLegacyMasterRoot`,
  `recoverRuntimeMasterRoot`, `decryptLegacyMasterRoot`. `openLegacyStore`,
  `closeRuntimeStores`, `closeStoresAfterError`,
  `isLegacyWalletAlreadyExists` and `runtimeStoreFactory` deleted.

Remaining on that branch:

- **Open design decision:** `Config.DB` still exists alongside
  `ManagerConfig.DB`. R6 wants the backend default on `ManagerConfig`, which
  means removing `Config.DB` — and that rewrites #1292, whose entire subject is
  the per-`Config` sqlite default. Decide before converting the call sites,
  because the two shapes imply different #1292 commits.
- 40 `NewManager()` call sites, 18 `Config{DB: …}` literals, bwtest, itest.
- `createRuntimeWallet` still opens its own store.
- **R4's aggregate transaction**: creation is still several writes, not one
  `walletdb.Update` with transaction-local ops and prepare-before-write.
- Then the full gauntlet and a split into reviewable commits across the three
  branches.

Verified: per-commit `go vet` across all 70 steps; six tag combinations; full
suite (minus `chain/`, which needs a live bitcoind and fails identically at
base); `-race` on `wallet/...` + `waddrmgr/...`; `fmt-check` and `sqlc-check`
in a throwaway worktree; golangci **v2.4.1** (CI-pinned) clean at all three
heads; every commit GPG-signed `%G? = G`.


**Branch:** `t313/1272` (19 commits, all GPG-signed), started from
`impl-secret-store` `0d98a7d45`. Verified green at each step: `go vet ./...`,
`go vet` under `itest dev`, `itest dev nolog` and `itest dev test_db_postgres`,
plus `go test ./wallet/... ./waddrmgr/...`.

**Done — sequencing step 1 (V1, V3, V5):**

| Item | Landed in | Notes |
| --- | --- | --- |
| **V1** ADR 0012 enforced | `wallet: route signer through Store` | waiver removed; `AddressType.IsScriptSpend` deleted with it (introduced by the same commit, no other caller); 3 unit cases replaced by rejection + watch-only mirror; taproot itest inverted and joined by the watch-only half it lacked |
| **V3** public `AddressInfo.HasScript` | `wallet: route signer through Store` | removed declaration + its sole write; the build confirmed nothing read it |
| **V5** `AccountSecret` → one field | `db: add account secret types`, `db: add GetAccountSecret to the store contract`, `wallet+db: add account secret adapters`, `wallet: route signer through Store` | six fields removed; `AccountSecretRowToSecret` collapsed to one copy; kvdb read reduced; signer reads the xpub via `GetAccount(SkipBalance)` |

Net vs `refs/backup/t313-green/impl-secret-store`: **12 files, +99 / −333.**

**Two things this surfaced, worth recording:**

- **V5 retires the F6 test-move.** The three `TestAccountSecretRowToSecret*`
  tests moved into `db: add account secret types` last round asserted overflow
  rejection on fields V5 deletes, so they are gone and that commit no longer
  carries a test file. The carry-forward entry for them is **moot** — superseded
  by V5, not dropped by accident.
- **Fixup targeting needed three attempts.** V5's test edits span four commits,
  and `buildAccountSecret` / `TestGetAccountSecretByID` live in later commits than
  the files' first appearance, so fixups aimed by "who created the file" landed
  wrong and conflicted. Aim them at *who introduced the specific lines*.

**Remaining:**

1. **#1272** — `HasAccountNumber` → `*uint32`; typed Vault methods (R3); delete
   F0/T7/`ScriptIsSecret`/public-passphrase fields; **the canonical
   single-encryption waddrmgr format at `mgrver` 9** (the largest single piece);
   ADR 0014; V4 PR-body correction. Note `wallet/deprecated*` is still at its
   round-1 delta (+240) — it returns to byte-identical when T7 goes.
2. **#1291** — V2 repeat-unlock revert; atomic aggregate creation on three
   backends; Manager-owned Store per R8; delete the teardown protocol and
   partial-create replay; kvdb single-wallet admission.
3. **#1292** — replay adapted to `ManagerConfig` / `Manager.Close`.
4. **Close-out** — per-commit `go vet`, four pinned lint passes at each of the
   three heads, `-race`, `fmt-check`/`sqlc-check`, both zero-touch checks,
   diff-accounting against `refs/backup/t313-green/*`, the 8 thread answers, both
   PR bodies, then push.

## Sequencing

1. **V1, V2, V3, V4, V5** — small, independent, correct regardless of encryption.
2. **ADR work** — draft 0014 for the single-encryption invariant. **ADR 0012 is
   left unchanged and enforced**; that is settled, not a decision to revisit.
3. **#1272 rebuilt** — `{Old, New}` passphrase API, single-encryption kvdb
   creation via `mgrver`, no dual machinery, field trims, ADR 0012 enforced.
4. **#1291 rebuilt** — atomic aggregate creation; Manager-owned Store with the
   R8 contract (one coarse mutex over Create/Load/publish/Close, restartable
   `Stop`, terminal fence at `Close`); repeat-unlock reverted; `RescanTarget` as
   its own commit.
5. **#1292 replayed** — its two *logical* commits kept, adapted to
   `ManagerConfig` and `Manager.Close`; not a byte-for-byte replay (R6).
6. **Diff against `refs/backup/t313-green/*`**; re-answer all 8 threads
   (including #1291's two on page 2); fix both PR bodies; push.

## References

- `.lgtm/tasks/{gpt,fable,codex}-review-task.md` — the three reviews.
- [[Task 311]] — the doctrine, incorporated here.
- [[Task 312]] — thread triage; re-dispositioned above.
- ADR 0001 — multi-wallet-per-database; evidence for C-bis.
- ADR 0009 / 0010 — the canonical vault model.
- ADR 0012 — enforced, not amended (V1).

## H3 — APPROVED (owner, 2026-07-30): canonical kvdb uses option (a)

**Decision.** kvdb wallet creation is **kept** — the kvdb itests create and
start wallets through `Manager.Create`, and the fixture should exercise the
go-forward format rather than a `Loader`-created legacy-dual wallet. New kvdb
wallets are created **single-encryption** per [[Task 311]].

**Approved: option (a)** (owner, explicit). Recorded with the rejected
alternative, because the kvdb wallet-secret store surface is `notImplemented`
today (`kvdb/walletstore.go:453-461`):

- **(a) waddrmgr grows a single-encryption format** — its canonical entry points
  write the **reserved next namespace `mgrver`**, and the generic migration path
  never advances a wallet to it. (Not a `latestMgrVersion` bump: that is exactly
  the R1 trap, where the credential-free engine stamps the new version onto a
  legacy-dual layout.) One master
  key derived from the private passphrase; public rows written plaintext per
  311 Option A; account/address secrets encrypted under keys enveloped by the
  single master (311 explicitly permits envelope internals). The kvdb vault
  becomes a thin single-passphrase adapter over this mode; the existing
  kvdb store adapters keep reading waddrmgr rows; the dual→single conversion
  migration targets the same format.
- **(b) canonical `WalletVault` over the kvdb store surface**: kvdb implements
  `GetWalletSecrets`/`UpdateWalletSecrets`/create-secrets plus bucket storage,
  and waddrmgr must stop encrypting internally for new wallets — deep waddrmgr
  surgery for a transitional backend.

**Now unanimous: (a).** gpt, fable and codex all recommend it. It is the
smallest format change satisfying the doctrine's contract (one passphrase, no
caller-selectable `CKTPublic`/`CKTScript` classes), it lands creation and the
future converter on the same format, and it avoids deep surgery in a backend
scheduled for the kvdb→SQL migration.

**But (a) is not a "thin adapter over current waddrmgr" — verified.** Current
watch-only waddrmgr wallets are structurally different from canonical watch-only
SQL wallets, so the new format has real work in it (gpt finding 2):

- `waddrmgr.Create` derives `isWatchingOnly := rootKey == nil`
  (`manager.go:2424`) and skips the private/script key creation under
  `if !isWatchingOnly` (`:2441`, `:2493`) — so a watch-only wallet has
  **no `masterKeyPriv` and no `cryptoKeyScript`**.
- `Lock`/`Unlock` reject watch-only managers with `ErrWatchingOnly`
  (`manager.go:1014`, `:1263`, `:1757`).
- Canonical SQL watch-only vaults *do* hold an unlockable script key
  (`vault_cipher.go:79-92`).

**The kvdb watch-only shape is not a new design — it mirrors SQL exactly**, which
already answers it (`keyvault/vault_genesis.go`, `CreateWalletSecrets`):

- `validateGenesisInputs` returns early for watch-only (`:118`), so the
  root-key-private and non-empty-passphrase checks are skipped — a watch-only
  wallet may have an **empty** passphrase;
- a master key is derived from that passphrase regardless;
- `cryptoKeyScript` is generated for **every** wallet (`:57`) and sealed under the
  master (`:63`);
- `if watchOnly { return }` (`:75`) — watch-only stops there;
- spendable wallets continue to `cryptoKeyPrivate` (`:81`) and the master HD
  private key sealed under it (`:98`).

So a watch-only wallet is: one passphrase (possibly empty) → one master key → one
script key; no private key, no master HD private key. It unlocks normally;
signing is impossible because the material is absent, not because a check forbids
it.

kvdb's single format does the same. Consequence worth stating: `Lock`/`Unlock`
begin working on watch-only kvdb managers — today they return `ErrWatchingOnly`
(`waddrmgr/manager.go:1014`, `:1263`, `:1757`) — because there is now a key worth
unlocking. Public metadata stays readable while locked (it is plaintext); script
bodies require an unlock.

**Correction — no daemon-itest gate** (codex correction 4). An earlier revision
of this section put `make itest db=kvdb` in the DoD, which contradicts the
preparation-only scope this round has held throughout. Require package/backend
coverage instead; see the Definition of Done below. The harness passing a single
passphrase (no `PubPassphrase`) stays required.

**Rebuild-guidance additions** (never-write list, alongside "dies
deliberately"): do not write `(*kvdb.Store).DB()` (312 H2 — dead on arrival),
and map `walletdb.ErrDbDoesNotExist` onto a `wallet/internal/db` sentinel from
the start instead of introducing `kvdb.IsMissingStore` (312 H1 — preserves the
walletdb import boundary). 309's field trims (candidate **3** only; 5 corrected
by V3, 2 superseded by V5, 1 and 4 deferred) are
build-time shapes for the new commits, not post-hoc edits.

Note on the line above, for consistency with the corrected field scope: of
309's candidates, only **3** is in scope this round, **5** is corrected by V3,
**2** is superseded by V5, and **1** and **4** are explicitly deferred.

## Implementation requirements

Added from the second review round. Each was verified against the code; these
are the gaps that would otherwise be discovered mid-rebuild.

### R1. `mgrver` needs format-aware entry points, not a new migration

The single most important finding of the round. `migration.Upgrade` runs
credential-free at `kvdb/lifecycle.go:98`, *before* `waddrmgr.Open` at `:103`,
and the generic engine stamps the version itself —
`mgr.SetVersion(ns, latestVersion)` at
`walletdb@v1.6.0/migration/manager.go:152`. `waddrmgr.Create` also writes
`latestMgrVersion` (`db.go:2637`).

So appending the single-encryption version to waddrmgr's migration list would
mark legacy-dual wallets converted **without rewriting any ciphertext**, and
bumping `latestMgrVersion` would make legacy creation stamp the new version onto
a dual layout. Full requirement and the required shape are recorded in
[[Task 311]] under "The `mgrver` trap".

### R2. Canonical passphrase API must be named explicitly

[[Task 300-INDEX]] says nothing from 305–312 executes unless this file names it,
and this file previously named only the *rotation* API. Naming the rest (gpt
finding 5):

- remove `PubPassphrase` from modern `wallet.Config`, `CreateWalletParams` and
  runtime kvdb configuration;
- rename `PrivatePassphrase` to `Passphrase` in the unmerged modern API;
- keep legacy public-passphrase arguments only on legacy waddrmgr / deprecated
  entry points;
- no backend-based public-passphrase branch anywhere in canonical wallet code.

### R3. `CryptoKeyType` must leave the canonical Vault contract

Deleting `ScriptIsSecret` and `scriptCryptoKeyType` is necessary but not
sufficient: `keyvault.Vault.Encrypt`/`Decrypt` still take
`waddrmgr.CryptoKeyType`, which keeps a legacy persistence enum on the canonical
boundary (gpt finding 6) — and `CKTPublic` is meaningless for SQL.

**DECIDED (owner): typed methods.** `EncryptPrivate`/`DecryptPrivate` and
`EncryptScript`/`DecryptScript`. No key-type parameter, no material-descriptor
type; wallet callers can no longer name a legacy crypto class. The option-(a)
adapter maps these fixed operations onto its internal waddrmgr implementation.

Note the distinction that keeps *some* split necessary: single **passphrase**
never meant single **key**. SQL still holds two enveloped secret keys —
`cryptoKeyPrivate` and `cryptoKeyScript` (`keyvault/vault.go:56`, `:59`) — under
one passphrase-derived master, so the vault must still know which sealed a blob.
Typed methods express that without leaking the enum.

**Recorded as a candidate follow-up, out of scope here:** collapsing to a single
secret key would remove even that distinction. `sql-wallet` is unmerged, so it
would need no data migration — only a schema edit to the two
`encrypted_crypto_key_*` columns. Deliberately deferred (codex scoped it out as a
lower-schema change).

### R4. Atomic kvdb creation needs a real transaction entry point

`kvdb.Store.CreateWallet` is `notImplemented` today
(`kvdb/walletstore.go:36-40`), and the kvdb creation path is
`CreateLegacyWallet` (`lifecycle.go:36`) — waddrmgr and wtxmgr are created
before any Store exists. Also, `waddrmgr.Create` performs scrypt, key generation
and HD derivation *inside* the caller's write transaction, which contradicts this
task's own prepare-before-write rule (gpt finding 3).

Required shape:

- make a kvdb Store/handle constructible over an **uninitialized** walletdb;
- prepare scrypt material, encrypted keys, default-account and imported-account
  inputs before the write, and zero the prepared plaintext after use;
- let `Store.CreateWallet` open **one** `walletdb.Update` creating namespaces,
  canonical waddrmgr state, wtxmgr state, default accounts, initial imported
  accounts and side metadata;
- use transaction-local ops for account writes — never call public Store methods
  that start a nested `walletdb.Update`;
- open runtime managers only after commit.

### R5. Wording: admission, not vault selection

Anywhere this plan or 311 says vault selection "keys on `mgrver`", read it as
(codex correction 2):

> Manager **admission** keys on `mgrver`; this round constructs only the
> canonical vault and refuses legacy-dual versions with
> `wallet.ErrMigrationRequired`.

Otherwise an implementer may rebuild the dual adapter the plan deletes. No
`LegacyManagerVault` is constructed by the modern runtime this round.

### R6. #1292 is logically unchanged, not byte-for-byte

Manager-owned configuration moves the SQLite default from per-wallet `Config` to
`ManagerConfig`, and harness cleanup must close the Manager after wallets stop.
So #1292 keeps its **two logical commits** — default new wallets to SQLite, adapt
the harness — but its code cannot be replayed verbatim. It must also use
`NewManager`, stop wallets, then close the Manager at the application boundary.

### R7. ADR 0014 lands in #1272 — small, extends 0009

Documentation inside the rebuild, not a decision. **Approved shape: a short,
cross-referential ADR that extends ADR 0009 rather than restating it.** ADR 0009
is already "Single-Passphrase Encryption Model" and already rejects the dual
hierarchy; what it lacks is scope, since its context is the SQL store. 0014 adds
only the new parts — all-backend scope, the kvdb `mgrver` mechanism, **kvdb's
one-wallet-per-database constraint**, legacy refusal with conversion deferred,
the plaintext/encrypted split, rejected Option B, and the two reconciliations
against ADR 0009:37 and ADR 0010. Full
contents in [[Task 311]].

Repo precedent is a new file, not an in-place amendment: ADR 0013 narrows 0012
explicitly. 0014 narrows 0012's kvdb grandfathering the same way. **ADR 0012
itself needs no edit** — this round enforces it.

### R8. Manager lifecycle contract — DECIDED

**Superseded shape removed.** An earlier revision of this section specified a
five-step admission barrier with an in-flight operation counter, an asynchronous
`Close(ctx)` and a cancellation/join protocol. That is over-built and is
**withdrawn**. A single coarse Manager mutex makes the counter unnecessary,
because `Close` acquiring the mutex already waits for any active `Create`/`Load`.

**The decided contract:**

```go
manager, err := wallet.NewManager(ctx, cfg)   // opens and owns one Store
if err != nil {
	return err
}
defer manager.Close()
```

- `NewManager(ctx, cfg) (*Manager, error)` opens and owns **one** Store for the
  configured database/network. The name is retained (owner direction).
- **One coarse mutex** serialises `Create`, `Load`, cache publication and `Close`
  admission. These are infrequent control-plane operations, so holding it across
  Store work and construction is acceptable — and it is what removes the need for
  any in-flight bookkeeping.
- `Manager.Close() error` — **synchronous, no context, idempotent, and it does
  not run wallet shutdown.** This reconciles the two reviews: gpt wanted `Close`
  to stop wallets, codex objected that `Wallet.Stop` needs a context a
  context-free `Close` cannot supply — passing `context.Background()` in a
  `defer manager.Close()` could block forever. codex's shape wins on that
  argument; gpt's terminal-fence concern is kept as step 2.
  1. lock the Manager (this waits for any active `Create`/`Load` to finish),
     mark it closed so later `Create`/`Load` return `ErrManagerClosed`;
  2. apply a **one-way terminal fence** to each cached wallet, so a caller
     holding a pointer cannot `Start` back onto a Store that is about to close.
     This is a flag, not a shutdown — it waits for nothing and needs no context;
  3. **unlock**;
  4. close the owned Store exactly once and return its result. Concurrent or
     repeated `Close` calls perform **one** physical close and observe the same
     stored result.

- **The caller stops and quiesces wallets before `Close`.** That is the
  documented contract. Calling wallet APIs concurrently with `Wallet.Stop` or
  `Manager.Close` is invalid use.
- `Wallet.Stop` stops only that wallet's workers and vault and **never closes the
  shared Store**.
- **`Wallet.Stop` is restartable** — decided; all three reviewers agree. The
  wallet stays cached, `Start` re-initialises its runtime, and `Manager.Load`
  returns the **same pointer**. The `stopped` terminal flag, `ErrWalletStopped`,
  `onStopped` and exact-instance eviction are all deleted.
- Wallets are restartable after a normal `Stop`, but **not** after
  `Manager.Close`.
- **No** `Manager.Stop`, `Manager.Unload`, per-wallet lease, in-flight admission
  counter, asynchronous close, or cancellation protocol.
- **No caller-owned public Store.** Considered and rejected on bounded-scope
  grounds: `db.Store` and the backend constructors are internal, so caller
  ownership needs a new public opaque Store API plus constructors — additional
  public surface that is deferred rather than forced into these three PRs. (The
  earlier "a store cannot stop the wallets" argument no longer distinguishes
  anything, since `Manager.Close` does not stop them either.) It can be added
  later if lnd needs to share an application pool.

**Why the deadlock disappears rather than being managed.** `w.onStopped` took the
Manager mutex (`manager.go:1181-1190`) and `Wallet.Stop` invoked it
(`controller.go:666`, `:715`), so any `Close` holding the mutex while waiting on
`Stop` would deadlock. Restartable `Stop` deletes `onStopped` outright, so no
Wallet→Manager re-entry remains. The rule still stands as an invariant: **wallet
code must never re-enter the Manager while the mutex is held.** `Close` never runs
wallet shutdown at all, and it releases the mutex before the physical Store
close.

**Constructor contract — now specified, not deferred.** `db.Store` has **no
`Close` method** (verified in `wallet/internal/db/interface.go`), so the Manager
holds an owned backend handle or an explicit closer.

For kvdb specifically (gpt finding 3, codex finding 4) — today `CreateStore`
creates legacy wallet namespaces immediately and `OpenStore` requires an already
initialised legacy wallet, so neither gives the pre-wallet Store that atomic
canonical creation needs:

- `NewManager` opens the existing backend **container**, creating an absent one
  as an **empty physical container** — no wallet namespaces, no manifest.
  Creating the container is not creating a wallet.
- `Manager.Create` performs the one aggregate wallet transaction.
- `Manager.Load` returns the documented not-found error on an empty container,
  `wallet.ErrMigrationRequired` for a legacy `mgrver`, and loads the canonical
  version otherwise.
- Constructor failure closes any partially opened backend handle.
- Canonical waddrmgr and wtxmgr runtime managers are opened only **after** the
  aggregate commit.
- **One Manager is assumed** (owner, 2026-07-30). Multiple Managers over one
  database are out of scope: not supported, not detected, not prevented. Build no
  guard, registry or cross-Manager coordination for it — the single-Manager
  assumption is the contract.

**In-flight public calls — decided, not left open.** An earlier revision offered
two alternatives; the choice is the caller contract above. Backend `Close` is
**not** a substitute: `sql.DB.Close` waits only for queries already started and
bbolt waits only for open transactions, while a single wallet method makes
*several* Store calls after one lifecycle check — `DecorateInputs` issues
separate `GetUtxo` and `GetTxDetail` calls, and a Store close can land between
them. So quiescing external calls is the caller's responsibility; `Close` drains
nothing. A drainable operation gate is the alternative only if `Close` must be
safe against arbitrary in-flight public methods, which this round does not
require.

### R9. kvdb is single-wallet — decided (owner, 2026-07-30)

kvdb keeps its legacy behaviour: **exactly one wallet per kvdb store.** No
multi-wallet bucket namespacing — that would be a large redesign of a
transitional backend and is out of scope.

This matters because the current adapter would silently allow otherwise, which
is verified: `Store.GetWallet` does not validate the requested name
(`kvdb/walletstore.go:67`), and `ListWallets` (`:133`) and `IterWallets`
(`:142`) both return `notImplemented`. So a Manager holding a per-name map over
a kvdb Store could publish two `Wallet` objects over one underlying wallet.

Required:

- a SQL-backed Manager admits multiple wallet names in one Store (ADR 0001);
- a kvdb-backed Manager admits exactly **one** wallet identity;
- a second name or alias is **rejected deterministically**, not opened over the
  same wallet;
- **no new persisted name metadata** — settled. Legacy kvdb has no durable
  wallet name; the database *file* is the wallet identity. Consequence accepted
  explicitly: the in-memory binding does **not** survive a restart, so after one
  any caller label opens that wallet. That matches legacy behaviour, and
  inventing durable name identity for a backend we are retiring is the wrong
  trade. Use a
  Manager-level singleton admission slot: the first successful `Create`/`Load`
  occupies it, a second `Create` returns the exists error, and another `Load`
  cannot publish a second `Wallet` over that Store. If differing caller labels
  must be rejected within one live Manager, bind the first label **in memory**.
  Reopening the same file under a different label is not a second wallet, which
  matches legacy behaviour;
- a package test proves **two `Wallet` objects cannot be published over one kvdb
  Store** — today's adapter accepts aliases silently, and even self-documents it:
  `GetWallet`'s NOTE says the supplied name "is not validated against the
  underlying wallet; the returned WalletInfo echoes the requested name".

The `NewManager` open-or-create contract for the kvdb container is specified in
R8; it is not restated here.

### R10. Two manager-version domains must not be conflated

Verified: SQL wallets are written with
`ManagerVersion: int32(waddrmgr.LatestMgrVersion)`
(`wallet/runtime_store.go:296`). So SQL and kvdb currently carry the *same
number* while meaning different things, and a uniform "canonical means version
N" admission test would either reject valid SQL wallets or admit legacy-dual
kvdb ones.

Define them separately:

- `mgrver` **inside the waddrmgr kvdb namespace** discriminates legacy-dual from
  the canonical single format;
- SQL `wallets.manager_version` stays ordinary manager-compatibility metadata and
  is **not** repurposed as an encryption-format marker.

The modern Manager still needs no `isKVDB` crypto branch: the kvdb opener checks
its namespace `mgrver` before migration/open and maps the legacy version to
`wallet.ErrMigrationRequired`; SQL opens its already-canonical representation. So
[[Task 311]]'s "never infer format from the backend" narrows to **never infer a
kvdb wallet's format from the configured backend alone.**

## Definition of Done

Consolidated, since the requirements were previously spread across the body,
§C-bis, H3, 311 and the index (gpt finding 9).

1. **Decisions recorded**: Manager-owned Store with the §C-bis API shape; kvdb
   format option (a) with R1's entry points.
2. **Modern API**: `{Old, New}` rotation only; `PubPassphrase` removed and
   `PrivatePassphrase` renamed per R2; the Vault exposes **typed methods**
   (`EncryptPrivate`/`DecryptPrivate`, `EncryptScript`/`DecryptScript`) and no
   `CryptoKeyType` anywhere on its contract, per R3.
3. **Admission**: legacy `mgrver` refused *before* construction with an
   exported, matchable `wallet.ErrMigrationRequired`; the credential-free
   migration runner can never advance a wallet to the single version.
4. **Watch-only single format**: canonical master for every wallet, enveloped
   script key, lock/unlock of that key, no private material ever; tests for
   empty and non-empty passphrases, locked metadata reads, locked script
   rejection, correct unlock, wrong passphrase.
5. **One aggregate transaction** on SQLite, PostgreSQL and canonical kvdb, with
   **failure injection after each durable step** proving a failed transaction
   leaves no wallet or account state, and no nested `walletdb.Update`.
6. **Store lifetime** (R8): two SQL wallets share one pool (ADR 0001); a
   kvdb-backed Manager admits exactly one wallet identity and rejects a second
   name deterministically (R9); same-name concurrent `Load` coalesces through the
   cache under the coarse mutex; `Create`/`Load` racing `Close` either completes
   or returns `ErrManagerClosed`; concurrent and repeated `Close` perform **one**
   physical Store close and return the same result; `Start` is refused after
   `Close` via the terminal fence. `Stop` is **restartable** — restart the same
   pointer, and `Load` returns that same pointer. No canceled-`Close`,
   retry-after-canceled-wait, or admission-barrier tests: those requirements were
   withdrawn with the asynchronous design.
7. **Field surface**: public `HasScript` and `ScriptIsSecret` gone;
   `AddressDerivation.Account` + `HasAccountNumber` **replaced by
   `AccountNumber *uint32`** (not merely deleted); `AccountSecret` reduced per
   V5. `Imported`/`Internal`/`Compressed` **untouched** — `AddressInfo.Imported`
   has two production readers (`address_manager.go:1138`, `signer.go:716`), so a
   review claim of zero readers for all three is wrong; removing them would be a
   behaviour change, not a dead-field deletion.
8. **ADR 0012 enforced**: spendable imports require `encrypted_priv_key`;
   script-only spendable tests removed.
9. **Verification**: per-commit `go vet`; four pinned golangci passes; `-race` on
   touched packages; `fmt-check`/`sqlc-check`; 80-col at tab-4; GPG-signed
   commits. **No daemon-itest gate** (R6/codex 4).
10. **Deprecated surface untouched** — both of these are **empty**, checked at
    the final #1272 head *and* again at the final stacked head:

    ```sh
    git diff sql-wallet..<head> -- 'wallet/deprecated*'
    git diff sql-wallet..<head> -- wallet/interface.go
    ```

    Zero-touch including bug fixes (owner); see [[Task 310]]. The two F2 defects
    there are permanent WONTFIX. **`LegacyManagerVault` is not rebuilt at all** —
    an earlier note here claimed base `deprecated.go` constructs it; verified
    false (zero `LegacyManagerVault` at `sql-wallet`, `kvdb/vault.go` absent
    there, and base `deprecated.go:7218` constructs `keyvault.NewWalletVault`).
    The adapter and its deprecated-file wiring were introduced by this stack and
    die with the dual machinery.
11. **Carry-forward accepted explicitly**, not just implied by range accounting:
    F1's accepted-request completion plus the unbuffered-`requestChan`
    invariant; `UfRny`, `UfRns`, `UfSB5`; per-test `testRootKey`; the moved
    conversion tests; exported `ErrInvalidPassphrase` and `TxStatus`;
    `RescanTarget` in its own commit; both #1292 logical commits; and a test
    proving a wrong **repeat** `Unlock` does not destroy an existing unlocked
    session or its timer.
12. **Migration out of scope** — no converter code, design or DoD item. Legacy
    wallets are refused by the modern `Manager`; the deprecated `Loader` keeps
    opening them unchanged (see [[Task 311]]).
13. **Bookkeeping**: all 8 review threads answered (including #1291's two on page
    2); both PR bodies corrected; range-diff/accounting against
    `refs/backup/t313-green/*` with every difference explained.

## kvdb is single-wallet (owner, 2026-07-30)

kvdb keeps its legacy one-wallet-per-database shape; multi-wallet is a SQL
capability only. Consequences for this plan:

- DoD 6's shared-pool / multi-wallet items are **SQL-scoped**, as written.
  Do not add kvdb equivalents.
- `Manager.Create` on a kvdb config whose database already holds a wallet
  refuses with the existing exists-error; no second-wallet admission path is
  built. Add one refusal test.
- The kvdb `db.Store` adapter keeps accepting `WalletID` nominally (contract
  shape) but implements no per-wallet keying, listing, or enumeration beyond
  the single wallet. Multi-wallet store surfaces stay `notImplemented` for
  kvdb, permanently.
- ADR 0014 states both kvdb constraints together: single encryption, single
  wallet.
