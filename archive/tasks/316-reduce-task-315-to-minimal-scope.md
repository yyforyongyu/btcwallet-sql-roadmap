# Task 316 — reduce Task 315 to minimal scope

> **CLOSED 2026-08-02 — archived.** The reduction executed: the surviving
> targeted-rescan identity redesign and the deferred/duplicate tests with their
> production support are gone, a base test rev 6 had deleted is restored, and
> every metric fell (all-file 16,645→15,269, production-Go 5,865→5,803,
> test-only 10,636→9,322, files 106→105) with the inventory gate at 7/43/43.
> Delivered and pushed at `c6425a509` / `df3a02daf` / `960dd02e8`.
>
> **§8's fixed 7/4/1 commit shape is superseded.** The owner relaxed the
> commit-weight cap to 300 rather than waiving it (`opus-report.md` §3.1), and
> seven of the twelve delivered commits exceed it. Task 324 owns the re-split.
> What survives from this file is the *final tree* at each PR boundary, which
> 324 must preserve byte-for-byte. Everything else here is evidence.

**Status:** **executed and pushed.** Final OIDs:
#1272 `c6425a509a4365822249c53e14433dace46389f8`,
#1291 `df3a02daf0d865265afef582f9b56c4d094f8334`,
#1292 `960dd02e871987daeacf03b69b0eaef0b46b749b` — 7/4/1, linear, all
`%G? = G`. The candidate inventory gate emits **7 / 43 / 43** and every §11
metric fell (all-file 16,645→15,269, production-Go 5,865→5,803, test-only
10,636→9,322, files 106→105). See `opus-report.md` §5 for the full gate
results, including the pre-existing environmental failures reproduced at the
pinned base, and §7.2 for the three post-push correction rounds — the last of
which made every commit, not just every PR head, compile and pass its tests.
Pushed to both remotes with every PR `headRefOid` matching; the
review-thread sweep is complete at 0 unresolved; and all three PR bodies are
rewritten to the delivered design (§12.2 step 6, snapshots at
`.lgtm/pivot/pr-bodies/`). Still open: green exact-head CI.

Section 1's document bootstrap is **complete**: Task 315 and `INDEX.md` were
updated on 2026-08-01 and name this file as the only plan to execute. Section
1's amendments stay binding. Task 315 remains the frozen governing requirement
set and the rev-6 input, and it is not complete until the PR bodies are
rewritten and CI is green at the OIDs above.

**Base pin:** `sql-wallet` =
`2ea76dfc5e85acf84014a21e4813cb23945250fa`.

**Rev-6 inputs — historical, pre-execution state (2026-08-01).** These are the
local heads this task consumed; they were rewritten in place by §9 and no
longer exist as branch tips:

- #1272, `impl-secret-store`, seven commits:
  `58d8d97549a6900926ea32bb6af05865971a5cbb`.
- #1291, `impl-runtime-store-6`, four own commits:
  `b6221ff78e156f10f2c8861a76a7dbddd891db61`.
- #1292, `wallet-default-sqlite`, one own commit:
  `70adc46faa8e275a56400a8c5bb09b4801d0c38d`.

The dependency chain is base -> #1272 -> #1291 -> #1292. At that time the
remote PRs pointed at the rev-5 heads
(`6eae9f519` / `61d2e60c1` / `4d672cbcf`) and no remote CI result validated
rev 6. Both statements are now obsolete: the PRs point at the executed OIDs in
the status paragraph above.

## 1. Authority and amendments

**Bootstrap: satisfied.** Task 315 identifies this file as the sole active
reduction plan and calls itself the frozen requirement set; `INDEX.md` opens
with "Execute Task 316 only" and its Fable row now says that file reviews Task
316 and spot-checks rev 6. Do not repeat these edits; keep them true as the
amendments below change.

Task 315 remains binding except for the five explicit amendments below.

1. The existing `manager create wallet` itest remains a simple create, start,
   and sync case. Add no case and no Unlock -> Stop -> Start -> Unlock sequence.
   The forced serial Stop-time Vault lock is proved at the narrow unit and
   Manager ownership boundaries, not by adding an itest scenario.
2. Mock-backed PSBT Store-routing coverage remains required. Real-backend,
   address-driven PSBT signing is deferred because the pinned base already has
   the non-mainnet purpose/coin mismatch recorded in `opus-report.md` section
   11.2 F1. Do not repair that defect here.
3. Preserve Task 315's exact-commit test requirement, but do not run its
   mutating `scripts/check-each-commit.sh` on final refs. Section 10.1 replaces
   that runner with a non-mutating exact-OID loop.
4. PostgreSQL is **compile fallout only**. This stack has no PostgreSQL goal,
   Manager, harness, daemon, runtime guarantee, or behavioral matrix, and no
   PostgreSQL row, case, or completion gate of its own. Task 321 owns real
   PostgreSQL support. What is forbidden is a **new** behavioral acceptance
   criterion, test case, row, or matrix for PostgreSQL; the repository's
   existing CI compatibility matrix stays mandatory. This amendment does
   **not** stop the existing shared DB contract from executing on PostgreSQL:
   `.github/workflows/main.yml:170-196` runs `itest-db` and `itest-db-race`
   over the `[sqlite, postgres]` matrix as a required job today, and this stack
   adds compile-forced pg Store methods that the suite exercises. Running it
   locally is CI parity and hygiene for compile-forced code, not a support
   claim — see section 10.2.
5. **Tests assert public behavior, not private structure** (owner ruling,
   2026-08-01), applying Task 315 §1's standing style rule to the Manager
   tests. Task 315 §5.1 phrases its kvdb invariant as "the rejected call does
   not replace the bound manager"; that describes the effect and does not
   mandate a private pointer assertion, and the public form is the stronger
   proof -- an unchanged `m.kvdbBound` does not show the first wallet still
   works, while reloading and using it does. Prove that invariant through
   subsequent public Manager and Wallet operations; see sections 4.5 and 5.4.
   This stack carries **no** Store-lifetime proof at the Manager boundary --
   `Wallet.Stop` has no Store-close path and harness teardown calls
   `Manager.Close` after Stop, which is sufficient here. This supersedes the
   earlier dispositions that retained `m.kvdbBound`, the private Manager-owned
   Store read, and `TestManagerBuildWalletFailureLeavesNoBinding`.

Tasks 305-314 stay archived. Tasks 317-323 stay deferred. Tasks 1000, 10001,
and 10002 are read-only reference material and are not implementation owners.
Do not edit them while executing this task.

## 2. Why this task exists

Rev 6 repaired the rev-5 correctness regressions and removed the live-watch and
deep-reorg additions, but it is not minimal:

- targeted rescans still reinterpret an account-number selector as Store
  `AccountID`, work Task 315 expressly defers;
- production support remains for a deferred Close race and fixture-only script
  modes;
- the itest contains lifecycle steps beyond the original scenario;
- stack-added tests repeat the same implementation path or test behavior owned
  by later tasks;
- three stale sync tests and the harness failed-Create test, plus several
  comments, describe behavior they no longer exercise;
- Task 315 still claims real-backend PSBT coverage that rev 6 correctly omits;
- Manager tests assert private structure -- `m.kvdbBound`, `m.cached`, a
  Manager-owned Store handle, direct `openKVDBBinding`/`buildWallet` calls --
  against Task 315 §1's standing rule.

This task removes those items. It adds no feature and fixes no pre-existing
behavioral defect.

## 3. Controlling minimality rules

Minimality is measured against the pinned `sql-wallet` base, not by deleting the
largest number of tests from rev 6.

Apply these rules in order:

1. Preserve an unchanged base test.
2. Minimally adapt a base test when a retained Store or Vault API forces it.
3. Remove or consolidate a stack-added test that repeats the same path and
   assertion.
4. Preserve separate tests when they can fail at different ownership or
   translation boundaries.
5. Remove production code added solely for a reverted or deferred test.
6. Do not move work between PRs to make one metric look smaller.
7. Do not create a helper, abstraction, shared test package, or table unless it
   is smaller than the code it replaces.
8. Do not chase a promised line count. Record the measured net result.

For every removed or consolidated test, record:

- whether it existed at the pinned base;
- the behavior it claimed;
- the retained test or reason the behavior is deferred;
- the net line change;
- the owning rewritten commit.

Where one production deletion removes a whole test file — `wallet/db_ops_test.go`
going with `wallet/db_ops.go` — every name still appears, but the reason, the
owning commit, and the net line change may be recorded once for the group.
Per-test line allocation inside a wholesale file deletion would be artificial.
That grouping never covers a test whose property outlived the deleted wrapper;
section 5.8 names those individually.

Review files are evidence, not executable addenda. GPT and Codex reviewed the
exact rev-6 heads. Fable later reviewed this task and spot-checked rev 6, but
its older duplication audit is no longer present in its current file. The
resolved dispositions in this task are authoritative; do not apply another
review item that is not carried below.

## 4. Production reductions

### 4.1 Remove targeted-rescan Store identity

Task 315 section 5.4 defers targeted-rescan identity redesign. The public
`[]waddrmgr.AccountScope` signature is not enough: its `Account` field must keep
base account-number semantics throughout the targeted path.

Remove from `wallet/syncer.go`:

- `errMissingStoreAccountID`;
- `scanSelectorKey` as a Store-ID selector type;
- `newResolvedScanTarget`;
- `storeAccountRecoveryID`;
- target-side `scanTarget.AccountID` and `scanTarget.info`;
- comments that call `AccountScope.Account` an `AccountID`;
- the target-side translation of a selector into a durable row identity.

**Do not remove targeted name resolution itself.** An earlier revision of this
section required `storeTargetedScanHorizons` to call `Store.GetAccount` with
wallet ID, scope, and account number, called `AccountName` metadata rather than
selector identity, and forbade an imported-account fallback. That is not base
behavior and it cannot work:

- `db.GetAccountWithOps` rejects every number lookup that resolves to an
  imported account -- "Imported accounts may only be looked up by Name" -- and
  that guard is **at the pinned base**
  (`wallet/internal/db/accountstore_getaccount.go:85-90` at `2ea76dfc5`);
- base therefore resolves each target's durable `AccountName` first and lets
  `storeScanHorizonAccount` prefer that name, falling back to the number only
  for a derived target;
- base proves this with a real-backend regression,
  `TestStoreScanHorizonsTargetedImportedNotResolvedAsDerived`
  (`2ea76dfc5:wallet/syncer_test.go:2367`), which targets the default derived
  account at number 0 **and** an imported-xpub account whose public number the
  Store masks to 0, and requires two distinct horizons;
- rev 6 deleted that base test and justified the loss with a `scanSelectorKey`
  comment claiming an imported-xpub account "cannot be targeted distinctly ...
  That is the base API's behavior". The claim is false: base can, and does.

Dropping base coverage is the opposite of minimality (section 3, rule 1), so
`resolveScanTargets` -- the plural, which **is** a base function -- stays in its
base role of filling `scanTarget.AccountName`. Its singular companion
`resolveScanTarget` is **not** base: rev 6 added it as the per-target Store-ID
matcher, so it is removed with the rest of that layer. What changes for the
plural is only its source: Task 315 forbids `walletdb.View` in wallet production
code, so instead of reading the legacy address manager inside a transaction, it
reads **one** `Store.ListAccounts` snapshot and builds a name index:

- **every backend:** index a non-imported row by its non-nil public
  `AccountNumber`;
- **kvdb only:** index an imported row by `db.AccountInfo.AccountID`, and only
  when the legacy address manager is present (`s.addrStore != nil`, the existing
  production boundary already used at `wallet/syncer.go:336`). kvdb's
  `AccountID` **is** waddrmgr's internal non-masked account number
  (`kvdb/accountstore_getaccount.go:279-290`) -- the same numeric selector base
  passed to `scopedMgr.AccountName`, so this is a format-boundary adaptation,
  not a public selector;
- **SQL:** never index an imported row for numeric target resolution. There
  `AccountID` is a relational row ID from `accounts.id` with no relation to a
  BIP44 number, so matching it would let a request for a nonexistent account
  select whichever imported row happened to share that integer. A numeric
  target that matches no derived row returns `db.ErrAccountNotFound`; SQL
  imported-account targeting has no base-compatible numeric selector and stays
  deferred;
- a derived mapping always wins if a malformed snapshot ever presents a
  collision;
- the matched row supplies `AccountName`, and nothing else from that row
  becomes a selector or a recovery key.

Key the index on the existing `waddrmgr.AccountScope`. This is a name resolver,
not a Store-ID selector: the identity leaves the resolver as a name, exactly as
at base, and section 4.2's private full-scan key still never becomes a public
targeted selector.

For the section 10.1 prohibited-symbol gate this means
`errMissingStoreAccountID`, `scanSelectorKey`, `resolveScanTarget`,
`newResolvedScanTarget`, `storeAccountRecoveryID`, `scanTarget.AccountID`, and
`scanTarget.info` must be absent from every applicable commit.

The SQL boundary is not a symbol-absence gate: `AccountID` remains valid in the
untargeted full-scan path. At every applicable candidate OID, run

```text
go test ./wallet -run '^TestResolveScanTargetsDoesNotUseSQLAccountID$'
```

and inspect `resolveScanTargets` at that exact OID. Require every targeted
resolver use of an imported row's `AccountID` as a lookup key to be
control-flow dominated by `s.addrStore != nil`; the `addrStore == nil` test
must return `db.ErrAccountNotFound`. This named test plus the exact-OID source
review is the gate that **no SQL path interprets `db.AccountInfo.AccountID` as
`waddrmgr.AccountScope.Account`**. Add no test for the deferred SQL behavior
itself.

Restore `scanTarget` to its base `{Scope, Account, AccountName}` shape and keep
base's `storeScanHorizonAccount` name-preferred lookup verbatim. The reserved
raw-import bucket is carried through with an empty `AccountName` and skipped
before any lookup, matching base behavior. A targeted derived account is
selected and keyed by its `(scope, account number)`; a targeted imported
account is selected by name and keyed by the non-masked number the caller
supplied.

Restore or minimally adapt these base tests rather than deleting them:

- `TestStoreScanHorizonsListAccounts`;
- `TestStoreScanHorizonsGetAccount`;
- `TestStoreScanHorizonsGetAccountKeepsImportedXpub`;
- `TestStoreScanHorizonsListAccountsKeepsImportedXpub`;
- `TestStoreScanHorizonsTargetedImportedBucketSkipped`;
- `TestStoreScanHorizonsTargetedImportedNotResolvedAsDerived`, the real-backend
  regression rev 6 deleted, together with its `createImportedXpubAccount`
  fixture. Adapt only what the resolver's new source forces -- it may no longer
  assert a walletdb read -- and keep both real-backend horizon assertions;
- targeted rows in `TestScanWithTargets_Errors`.

Delete these two stack-added structural tests that pin the deferred model:

- `TestInitializeCarriesStableAccountID`;
- `TestResolveScanTargetsUnknownAccountID`.

Repurpose the third, `TestResolveScanTargetsStoreOnlyAccount`, in place rather
than deleting it -- its footprint becomes the backend-boundary regression. As a
small mock test with `addrStore == nil`, it returns one imported row whose
`AccountID` equals the target number and whose `AccountNumber` does not, then
requires `db.ErrAccountNotFound`. Its final name is
**`TestResolveScanTargetsDoesNotUseSQLAccountID`**. Add no second
resolver test and no real-backend SQL suite: this proves the new branch without
claiming SQLite imported-account targeting is supported.

Removing `errMissingStoreAccountID` does not remove the case it guarded: the
untargeted full scan keys on the Store row ID, so a nil `AccountInfo.AccountID`
must still fail rather than silently key on zero. Return a plain wrapped
`fmt.Errorf` naming the account; `err113` is disabled repo-wide
(`.golangci.yml:45`), so no replacement sentinel is needed and none may be
introduced under a new name.

Rewrite `TestRecoveryStateKeysByStoreAccountID` as an explicitly untargeted
full-scan collision regression under the name
**`TestFullScanRecoverySeparatesStoreIDAndAccountNumber`**. It must
not call or assert a targeted selector. Delete `seedScanStateAccountID` or an
equivalent identity-only fixture only if the retained batch/horizon tests no
longer need it.

### 4.2 Preserve only required SQL recovery identity

Do not interpret the previous section as a repo-wide ban on `AccountID`.
`db.AccountInfo.AccountID` and `db.ScanHorizon.AccountID` remain valid Store
metadata. SQL uses the latter to persist an account horizon independently of
renames.

The final recovery rules are:

- targeted state keys derived accounts by account number;
- untargeted Store full-scan state may key accounts by Store ID so an imported
  account ID cannot collide with another account's BIP44 number;
- a Store-backed deriver retains the real `AccountNumber` separately from that
  private recovery key;
- the deriver passes the real account number to
  `AddressDerivationParams.DerivedAccountNumber` for a derived account and nil
  for an imported account;
- the private full-scan key must never become a public targeted selector;
- horizon persistence may carry Store ID separately from the targeted key;
- comments must not claim Store ID and account number are interchangeable.

This split needs a carrier that `waddrmgr.AccountProperties` does not provide:
it holds exactly one number field, which becomes the recovery key. Add a
wallet-local snapshot alongside it — the horizon props, the Store row ID, and
the real BIP44 account number (nil for imported) — and register the row ID and
the store-native deriver on the recovery state before `Initialize` runs, since
initialization derives the lookahead. Do not widen `AccountProperties` and do
not change `Initialize`'s base signature.

Keep the Store-backed full-scan reads, account xpub/schema conversion,
`recoveryAddressDeriver`, `storeRecoveryDeriver`, and SQL horizon persistence.
Keep focused proof in:

- `TestInitializeUsesStoreRecoveryDeriver`;
- `TestDeriveStoreAddressEncodesTypes`;
- `TestScanHorizonParamsStampsAccountIdentity`;
- shared `TestApplyScanBatchResolvesHorizonByAccountID`.

Use unequal values, for example account number 7 and Store ID 31. The focused
tests collectively prove the split:

- `TestStoreScanHorizonsGetAccount` proves target selection remains account 7;
- recovery-deriver tests prove derived-address parameters receive account 7,
  not 31, and an imported account receives nil;
- `TestScanHorizonParamsStampsAccountIdentity` proves the emitted horizon
  carries Store ID 31;
- shared `TestApplyScanBatchResolvesHorizonByAccountID` proves the SQL adapter
  advances row 31;
- the renamed untargeted full-scan collision test proves imported Store ID 3
  and derived account number 3 cannot overwrite one another.

The Manager itests are lifecycle and routing smoke tests. They do not prove SQL
lookahead because the SQLite case starts with no accounts.

### 4.3 Narrow the waddrmgr account-secret accessor

The new `ScopedKeyManager.AccountSecret` currently decrypts and returns account
public metadata, fingerprint, name, and imported state. Its sole production
consumer discards all of those values and needs only the persisted encrypted
account private key.

In the commit that introduces the account-secret Store query:

- make the accessor return only a cloned encrypted private-key blob and error;
- retain nil for watch-only, imported, or otherwise private-less accounts;
- retain account-not-found translation;
- remove `decryptAccountPubKey` if it becomes unused;
- remove the root-Manager lock and Close-race commentary added solely to make
  concurrent use-after-Close safe;
- simplify `accountSecretReader` and the kvdb adapter accordingly;
- update the focused waddrmgr tests to the narrower contract.

Task 319 owns Manager Close races. This task keeps ciphertext copy safety for
normal reads but does not harden calls racing Close.

### 4.4 Remove unused script-import generality

In the kvdb address bridge, remove any new `secretScript bool` parameter from
P2WSH/taproot import helpers when every retained production caller passes
`false`. Restore the base hard-coded non-secret shape.

Keep all of the following:

- `db.AddressSecret.ScriptIsSecret`;
- the waddrmgr per-row script-secrecy bit;
- P2SH synthesis as secret;
- non-secret taproot persistence and locked readback;
- `LegacyWalletVault.Lock` treating `waddrmgr.ErrWatchingOnly` as a no-op;
- SQL's `CKTPublic` to script-key mapping.

Task 323 owns an old-wallet secret-witness fixture. Removing that fixture does
not authorize removing production provenance for existing rows.

### 4.5 Keep only the forced serial Stop repair

Keep the unconditional `w.keyVault.Lock()` after workers stop and before the
wallet enters `Stopped`. It prevents a vault-routed Unlock from leaving runtime
keys alive across a serial Stop.

Remove:

- `TestStopLocksVaultWhenStateBitCleared`;
- `createStartedWalletForVaultRace`;
- Unlock/Stop race commentary;
- the Unlock -> Stop -> Start -> Unlock steps from `itest/manager_test.go`.

Make the base `TestControllerStop` place the wallet in the unlocked state,
require exactly one Vault `Lock`, and assert the stopped state. Then delete the
stack-added `TestControllerStopLocksVaultWhenUnlocked`. That adapted base test
is the **sole** Stop-to-Vault dispatch proof.

**Delete `TestManagerStopClosesNoStore` entirely.** Earlier revisions of this
section tried to keep it, first by fixing its comments, then by reducing it to
one start, one `Wallet.Stop`, and a cached `Manager.Load`. Once it stopped
claiming Store lifetime, nothing distinct was left:

- `Manager.Load` returns the cached pointer without touching the Store
  (`wallet/manager.go:564-569`), so the assertion would also pass over a closed
  Store -- it never was a Store-lifetime proof;
- `Wallet` holds no Manager back-reference, so `Wallet.Stop` structurally cannot
  evict `Manager.wallets`. The assertion has no falsifier this stack can reach;
- what remains is base `TestControllerStop` behavior composed with base
  `TestManagerLoad_ExistingWallet` behavior, both already retained.

So remove it from Task 315 §5.5 and DoD 6, from sections 5.4, 6.3, 10.3 and 14
here, and from the report's falsifier map. Keep the adapted base
`TestControllerStop` as the sole Stop-to-Vault dispatch proof, base cached-Load
identity plus the reduced SQLite cached-Load case for cache retention, and
harness teardown as the one exercised Stop-then-`Manager.Close` sequence. Add no
replacement restart, Store-handle, or Manager lifecycle test: deleting is
smaller from `sql-wallet` and follows the owner's lifecycle deferral.

The existing itest case must end after create, start, mine, and sync. Harness
teardown owns Stop and Manager Close.

### 4.6 No new PostgreSQL or generated scope

Task 316 adds no PostgreSQL Manager, harness, daemon, or behavioral matrix.
Retain only compile-forced PostgreSQL Store methods and the already shared DB
contract created by Task 315's Store interface.

No SQL query, schema, migration, proto, module, sqlc output, or protobuf output
change is authorized relative to the rev-6 inputs. Never edit generated files
manually. If execution appears to require such a change, stop and re-check the
scope instead of regenerating unrelated output.

## 5. Exact test disposition

### 5.1 Disposition deferred or fabricated cases

All names below are file-qualified where a same-named test exists elsewhere.

- Delete `waddrmgr/manager_secret_test.go::TestAccountSecretRacesClose`.
- Delete `wallet/controller_test.go::TestStopLocksVaultWhenStateBitCleared`
  and `createStartedWalletForVaultRace`.
- Delete only
  `wallet/internal/db/kvdb/addressstore_getsecret_test.go::`
  `TestNewImportedTaprootScriptWatchOnly`.
- Keep the same-named shared SQL Store test and
  `TestScriptForOutputWatchOnlyTaprootSQL`.
- Delete kvdb `TestNewImportedScriptTransportKeyIsPublic`.
- Delete kvdb `TestGetAddressSecretSecretWitnessScript`.
- Retain a minimized, non-parallel `TestLegacyWalletVaultLockWatchOnly` because
  it is the focused proof for the required `ErrWatchingOnly` no-op. Do not
  mutate package-global logging from a parallel test.
- Delete `TestEncryptTaprootScriptModeIndependent`.
- Delete kvdb `TestGetAddressSecretImportedScript` only after
  `TestImportTaprootScriptKVDBReadableWhileLocked` asserts ciphertext,
  `ScriptIsSecret=false`, locked readback, and `CKTPublic` decryption.
- Delete `TestHandleUnlockReqRepeatDoesNotLock` and add **no** replacement
  row. `handleUnlockReq` differs from base only by routing `DBUnlock` to
  `keyVault.Unlock`, and every Unlock error takes the one base error-return
  branch that base `TestHandleUnlockReq_Errors/DBUnlock_Failure` already
  proves — minimally adapted to expect the Vault call. The already-unlocked
  contract is base's and is owned by base
  `TestWalletVaultUnlockAlreadyUnlockedReturnsError`; a mocked "wrong
  passphrase while already unlocked" cannot occur through the real
  `WalletVault`, which returns `ErrVaultUnlocked` before it validates. No
  retained production change adds a lock on that path.

Replace `TestLegacyWalletVaultScriptKeyFollowsWalletMode` with one small table
containing exactly two explicit-key forwarding rows: `CKTPublic` and
`CKTScript`. The Vault does not choose a key based on wallet mode, so remove
that policy claim and retain no watch-only/spendable mode matrix in this test.

### 5.2 Preserve base tests

Do not delete or merge these merely because rev-6 setup now makes them look
similar:

- SQL `TestGetAddressSecret` and positive `TestNewImportedAddress` rows;
- `TestRequireAddressPrivKeyOnSpendable`;
- `TestNewAddressOnSQLOnlyAccount` and
  `TestGetPrivKeyForAddressSQLDerivedAddress`;
- `TestDerivePrivKeyFails`, `TestDerivePrivKeyFail`, `TestECDHFails`, and
  `TestECDHFail`;
- `TestManagerLoad_ExistingWallet`;
- `TestControllerChangePassphrase`;
- external and internal `GetDerivationInfo` tests;
- `TestCheckRollbackStoreSyncedTip` and `TestCheckRollbackDetected`;
- `TestAdvanceChainSyncStoreSyncedTip` and
  `TestAdvanceChainSyncUsesStoreSyncedTo`;
- `TestScanWithRewindRoutesStoreRewind` and
  `TestScanWithRewindStoreSyncedTip`;
- `TestProcessChainUpdate_Disconnect`, `TestProcessChainUpdate`, and the
  dedicated connected/disconnected routing tests;
- `TestStoreScanHorizonsGetAccount`, the restored
  `TestStoreScanHorizonsTargetedImportedNotResolvedAsDerived`, and the other
  base scan-horizon tests.

Restore their closest base-shaped form compatible with retained Store routing.
For `TestRequireAddressPrivKeyOnSpendable`, keep the base three distinct inputs
and remove the added public-only/script-only duplicate rows that call the helper
with identical arguments.

### 5.3 Consolidate stack-added exact duplicates

Apply these exact choices:

- Delete `wallet/runtime_create_test.go`. Keep
  `TestRuntimeCreateWalletParamsCreatesSpendableSecrets`, including
  `MasterKeyPrivParams`, and keep the separate birthday-verbatim test.
- Delete `TestChangePassphraseWrongOldKeepsUnlockedRealVault`. The keyvault
  wrong-old state test and the public real-backend
  `TestWalletChangePassphraseMatrix` own the two distinct boundaries. Add no
  new public case solely to replace this test -- in particular do not adopt the
  review suggestion to move its assertions into that matrix, because section
  5.2 preserves base `TestControllerChangePassphrase` instead.
- Consolidate the four `TestLegacyWalletVaultChangePassphrase*` tests into one
  table retaining private-only, public-only, combined, and wrong-public adapter
  behavior.
- Fold the three standalone waddrmgr ciphertext-copy tests --
  `TestAccountSecretReturnsCopy`, `TestManagedAddressSecretPrivKeyCopy`, and
  `TestManagedAddressSecretScriptCopy` -- into `TestAccountSecretDerived`,
  `TestManagedAddressSecretPubKey`, and `TestManagedAddressSecretScript`,
  carrying the `assertCiphertextCopy` helper with them and deleting it if it
  ends up with a single caller.
- Delete `TestManagerLoadUsesRuntimeStore`.
- Delete obsolete `TestManagerLoadClosesRuntimeStoreOnMetadataError`; it no
  longer tests a close, and its privately injected malformed-metadata assembly
  is outside this stack. Do not claim another retained test covers it -- after
  section 5.4 the post-binding assembly test is gone too.
- Rewrite `TestOpenRuntimeStoreSQLiteRequiresWalletRow` through public
  `Manager.Load`, not private `openRuntimeStore`.
- Delete `createRuntimeStoreTestWallet`, `expectRuntimeWalletRow`, and
  `expectRuntimeWalletRowRaw` after their final caller is gone.
- Rename `TestManagerCreateLoadSQLiteNoLegacySidecar` to
  **`TestManagerSQLiteCreateLoadCached`** and retain only what that name
  claims: public SQLite `Create` plus same-Manager cached `Load` pointer
  identity. Its private nil-sidecar assertions go with §5.4's rule, and its
  derivation/signing tail repeats stronger tests. Carry the new name into the
  ledger, section 8's commit ownership, Task 315's SQLite DoD row, and
  `opus-report.md`.
- Remove the cached-Load segment from
  `TestManagerIgnoresPerWalletDBAndChainParams`; retain fresh-Manager Load on
  both backends.
- Delete `TestResolveDerivedPrivKeyFromStoreRejectsWatchOnly`,
  `TestResolveDerivedPrivKeyFromStoreAbsentAccount`, and
  `TestCanUseAddressInfoDerivation` after mapping their properties to the
  public signer and imported-xpub tests.
- Keep `TestResolveDerivedPrivKeyFromStoreDerivedSuccess` only as the focused
  marked-ciphertext decrypt/derive boundary.
- Fold `TestSignPsbtResolvesNonZeroInternalAccount` into account 0/non-zero
  rows in base `TestSignPsbt`.
- Table-drive `TestScriptForOutputP2SH`, `TestScriptForOutputP2WSH`, and
  `TestScriptForOutputTaprootScript` through their shared assertion path. Keep
  real SQL watch-only and real kvdb locked tests standalone.
- Remove `dbScopeAddrSchema` and `TestDBScopeAddrSchemaMapsTypes` if no retained
  production caller remains.
- Delete tautological `TestStoreRecoveryDeriverMatchesAddressData`; keep the
  independently constructed address-encoding test.
- Reduce `TestRecoveryAddrTypeMapsScopeSchema` to one external case, one
  internal case, and invalid branch.
- Table-drive the spendable, watch-only, and wrong-passphrase
  `CreateWalletSecrets` cases. Keep input guards, ciphertext-only assertions,
  and production-scrypt coverage.

### 5.4 Manager tests that must remain distinct

Keep all of the following:

- `TestManagerCreateFailureLeavesManagerReusable`;
- `TestTeardownWalletsClosesManagerAfterFailedCreate`;
- `TestManagerCreateUsesCommittedWalletRow`;
- pre-binding Create failure coverage;
- base cached-Load identity and fresh-Manager reopen coverage.

The direct Manager failed-Create test proves reuse and publication behavior.
The harness test proves registration, teardown, and database release. They are
not duplicates.

**Assert public behavior, not private structure** (Task 315 §1, owner ruling
2026-08-01, amendment 5). Task 315 §5.1 states its kvdb invariant in
implementation terms -- "the rejected call does not replace the bound manager"
-- but that is a statement about the effect, not a mandate to assert a private
pointer, and the public form is the stronger proof: an unchanged
`m.kvdbBound` does not show the first wallet still works, whereas reloading and
using it does. Apply that throughout the Manager tests:

- `TestManagerKVDBRejectsSecondName`: require a second-name `Create` **and**
  `Load` to fail, then reload the first name, place it in the existing
  test-started setup, and call public `ListAccounts` -- that one named
  operation is the whole "still usable" proof. Remove the `m.kvdbBound` pointer
  assertion;
- delete `TestManagerBuildWalletFailureLeavesNoBinding`. It drives
  `openKVDBBinding` and `buildWallet` directly and inspects `m.kvdbBound` to
  inject a private post-binding state the public goal does not require; the
  corrected-`Create` retry test owns failure reuse. This reverses the earlier
  keep-it disposition, which predates the owner ruling;
- remove the private `m.cached` assertion from
  `TestManagerCreateFailureLeavesManagerReusable`; the successful retry on the
  same Manager is already the stronger public proof;
- remove `require.Same(store, w.store)` from
  `TestManagerCreateUsesCommittedWalletRow`. Its strict mock plus public
  `w.ID()` already proves Create used the committed row;
- remove the `w.addrStore`/`w.txStore` sidecar assertions from the renamed
  `TestManagerSQLiteCreateLoadCached` (§5.3) and the `w.addrStore` assertion
  from `TestManagerSQLiteReopenDerivesAddress`. Cached identity belongs to the
  first; fresh-Manager `Load` and public `NewAddress` routing belong to the
  second, which already proves it without touching a private field;
- reduce `TestManagerIgnoresPerWalletDBAndChainParams` to public `Create` and
  fresh-Manager `Load` success with both per-Wallet fields nil; remove the
  `m.chainParams`, `w.cfg.ChainParams`, and `w.cfg.DB` assertions.

Scope this rule to **stack-added or materially rewritten** test bodies. Do not
churn an unchanged base test that already reaches package-private setup.

Keep `TestManagerCreateUsesCommittedWalletRow` and the pre-binding Create
failure coverage: with the assertions above removed they observe caller-visible
outcomes only.

### 5.5 Signer and PSBT survivors

Keep `TestSigningAcrossBackends` for path-only APIs on both real backends. Keep
its address-driven tail only for kvdb; the base
`TestComputeUnlockingScriptSQLDerivedAddress` remains the SQL address-driven
proof. Both must execute and validate the produced unlocking script.

Keep these separate regressions:

- `TestGetPrivKeyForAddressKVDBP2PK` for derived P2PK normalization;
- `TestOpenWithRetryRoutedSigning` for deprecated-constructor imported-WIF
  routing and constructor wiring;
- public imported-xpub refusal;
- mock-backed `TestSignPsbt` with zero and non-zero accounts.

Do not add real-backend PSBT coverage or repair persisted-scope resolution.

### 5.6 Cross-layer overlap that must remain

Do not de-duplicate across these boundaries:

- waddrmgr primitive accessors, kvdb Store adapters, shared SQL Store
  conformance, and public Wallet behavior;
- common query validation (`TestGetAddressSecretQueryValidate`), one kvdb
  adapter rejection, and the existing shared SQL adapter rejection -- these
  prove the retained adapters actually call `Validate`;
- the shared DB contract's source, which both the SQLite and the PostgreSQL
  test binaries execute; do not delete a shared case merely because PostgreSQL
  also runs it, and do not add a PostgreSQL-only row;
- SQL keyvault persisted/failure-state tests and public Wallet rotation;
- Legacy Vault adapter translation and public Wallet rotation;
- cached Load identity and fresh-Manager reopen;
- path-only signer APIs and address-driven script construction;
- Manager failure recovery and harness teardown;
- pre-binding Create failure, and the exact committed-row regression
  `TestManagerCreateUsesCommittedWalletRow` -- it prevents a post-commit
  re-query failure and is not a generic post-commit failure suite. Post-binding
  assembly failure is **not** retained: section 5.4 deletes its only test as a
  privately injected state, and section 5.4's named list controls.

Also retain both `TestGetAccountSecretCanceledCtx` and
`TestGetAddressSecretCanceledCtx`. They cover separate Store entry points;
sharing an internal context check does not prove each adapter invokes it.

In particular, retain keyvault tests for locked rotation, wrong old
passphrase, validate-before-mutate, Store update failure, persisted success, HD
root preservation/corruption, and watch-only material. The public matrix cannot
observe their exact persisted and in-memory failure states.

### 5.7 Fixture reductions

Apply only fixture changes made safe by the retained tests:

- remove the unused real `setupTestDB` call from
  `createTestWalletWithMocks` if no base test still reads it;
- remove its duplicate `mockVault.AssertExpectations` call;
- inline and delete single-use `newTeardownHarness`;
- remove runtime-store mock fixtures after their callers are removed;
- simplify stack-added `testKVDBPath` cleanup to rely on `t.TempDir` only if
  every caller can be updated without touching unrelated base behavior.

Do not add a cross-package helper merely to save setup lines.

### 5.8 Measured base-test deletion ledger

Sections 5.1-5.7 disposition the tests the reviews named. That is not the same
as dispositioning every base test this stack removes, and the difference is
where the round-4 blocker hid. Section 10.2's gate, run with the **serialized**
generator against the recorded **rev-6 inputs**, gives the authoritative audit:
**11** base test declarations absent at #1272 and **51** at both #1291 and
#1292. Reproduce those before executing; a different number means the generator
was not pinned.

Those are input-audit sizes, **not** the expected candidate output. This plan
restores eight of the deleted declarations under their base `path::TestName`:

- at #1272, `TestHandleLockReq_Errors` and the three
  `TestControllerChangePassphrase_Interrupted_Wait*` declarations;
- at #1291, inherited by #1292,
  `TestControllerStart_DBGetAllAccountsFail`, the minimally adapted
  `TestDerivePubKeyDeriveFails`,
  `TestStoreScanHorizonsTargetedImportedNotResolvedAsDerived`, and the minimally
  adapted `TestStampRecoveryAccountIDsCarriesStableID`.

A correct rebuild therefore emits **7** at #1272 and **43** at #1291 and #1292.
A candidate run that still emits 11/51/51 proves the restorations did not
happen. Only an additional same-name restoration discovered while rebuilding —
documented when it is made — may lower those numbers.

| file | count | disposition |
| --- | ---: | --- |
| `wallet/db_ops_test.go` | 37 | Three distinct owners, not one. **27** ordinary declarations go with the wholesale deletion of `wallet/db_ops.go` at #1291 (§3 permits one reason, one owning commit, and group-level net lines for them). **3** — `TestDBUnlock`, `TestDBPutPassphrase`, `TestDBPutPassphrase_Error` — die earlier, at **#1272**: their subjects `DBUnlock` and `DBPutPassphrase` (base `db_ops.go:206` and `:222`) are already absent from `db_ops.go` at `58d8d9754`, removed with the Vault routing, so they map to the retained handler and Vault coverage of that routing, not to a file deletion. The remaining **7** kept their property when the wrapper went — see the second table |
| `wallet/controller_test.go` | 7 | Restore `TestHandleLockReq_Errors` unchanged: its stopped-state branch never reaches the replaced lock dependency, and renaming it only to remove an underscore is unrelated churn against section 3 rule 1. `TestHandleLockReq_Idempotency` maps to `TestWalletVaultLockIdempotent`, which owns idempotent locking at the Vault boundary. `TestHandleLockReq_LockError` is **removed with its branch**: `keyvault.Vault.Lock()` is void at the head (`keyvault.go:35`), so the error return it asserted no longer exists. Restore `TestControllerStart_DBGetAllAccountsFail` under its base name: at the pinned base its body already drives `Store.ListAccounts`, and rev 6 changes only its name/comment to repair pre-existing terminology, which this reduction does not own. The three `TestControllerChangePassphrase_Interrupted_{WaitCancelled,WaitShutdown,WaitTimeout}` are **restored unchanged**: rev 6 keeps the same `sendReq` → `waitForResp` path in `wallet/controller.go`, so all three branches still exist, and keeping the base tests is smaller from `sql-wallet` than deleting and justifying them |
| `wallet/signer_test.go` | 4 | Restore and minimally adapt `TestDerivePubKeyDeriveFails`: return a valid account xpub from the Store, use `hdkeychain.HardenedKeyStart` for the branch or index, and require the public-child derivation error to propagate. The retained Store-path set otherwise has no falsifier for `deriveStoredAccountChildPubKey`'s branch/index error path. `TestDerivePubKeyFetchManagerFails` maps to `TestDerivePubKeyStoreFails`, the replacement dependency's unexpected-read error. `TestDerivePubKeyNotPubKeyAddr` is removed with the managed-address/type-assertion path; `TestDerivePubKeyMissingAccountPubKey` separately owns missing Store public metadata. `TestResolvePrivKeyFallsBackAfterCacheMiss` is **removed with its production path** and must **not** be restored: it guarded `resolvePrivKey` → `resolveDerivedPathPrivKey` including the `DeriveFromKeyPathCache`/`ErrAccountNotCached` branch, and both functions are gone from `wallet/signer.go` at the head (2 → 0; the remaining `DeriveFromKeyPathCache` hits are `deprecated.go` only). Cite `TestResolveDerivedPrivKeyFromStoreDerivedSuccess` as proof of the new Store-backed path, never as proof the cache fallback survived |
| `wallet/syncer_test.go` | 2 | `TestStoreScanHorizonsTargetedImportedNotResolvedAsDerived` is restored by section 4.1. `TestStampRecoveryAccountIDsCarriesStableID` is **minimally adapted** to section 4.2's retained full-scan registration path — not optional: it is the base falsifier for stamping a durable Store ID onto recovery state, and `stampRecoveryAccountIDs` went from 4 production hits at base to 0 at the head with no replacement test |
| `wallet/internal/keyvault/vault_lifecycle_test.go` | 1 | `TestWalletVaultChangePassphraseLockedRequiresUnlockedState` asserts the **opposite** of Task 315 §5.4's ruling that rotation must work while locked; the head's `TestWalletVaultChangePassphraseLockedRotates` replaces it. Record it as a deliberate behavior reversal, not a de-duplication |

The twenty-seven ordinary `db_ops` declarations discharged by the single
disposition "removed with `wallet/db_ops.go`" and one owning #1291 commit:

```text
TestDBBirthdayBlock                TestDBCreateWallet
TestDBDeleteExpiredLockedOutputs   TestDBGetAllAccounts_Error
TestDBGetScanData                  TestDBGetScanData_AddressError
TestDBGetScanData_Error            TestDBGetScanData_ManagerError
TestDBGetScanData_UTXOError        TestDBGetSyncedBlocks
TestDBLoadWallet                   TestDBPutBirthdayBlock_Error
TestDBPutBlocks                    TestDBPutBlocks_Error
TestDBPutRewind                    TestDBPutRewind_Error
TestDBPutSyncBatch_Error           TestDBPutSyncTip_Error
TestDBPutTargetedBatch_Errors      TestDBPutTargetedBatch_WithTxns
TestDBPutTxns                      TestDBPutTxns_AddressNotFound
TestDBPutTxns_Error                TestDBPutTxns_UnconfirmedError
TestPutAddrHorizons                TestPutAddrHorizons_Error
TestPutSyncTip_Error
```

The three that fall inside #1272's input audit are dispositioned separately,
because their production subjects go with the Vault routing rather than the
file:

| base test | disposition |
| --- | --- |
| `TestDBUnlock` | subject `Wallet.DBUnlock` removed at #1272; unlocking is routed to `keyVault.Unlock`, owned by the adapted base `TestHandleUnlockReq_Errors` and the keyvault lifecycle tests |
| `TestDBPutPassphrase` | subject `Wallet.DBPutPassphrase` removed at #1272; rotation is routed to the Vault, owned by `TestControllerChangePassphrase` and the keyvault rotation set |
| `TestDBPutPassphrase_Error` | same subject and same owners, error branch |

The seven `db_ops` tests whose property outlived their wrapper:

| base test | disposition |
| --- | --- |
| `TestLoadWalletScanDataKeepsImportedXpubHorizons` | retained as `TestStoreScanHorizonsListAccountsKeepsImportedXpub` |
| `TestPutAddrHorizonsEvictsOnPostExtensionReadError` | retained as `TestApplyScanBatchEvictsHorizonOnPostExtensionReadError` |
| `TestDBPutSyncBatchEvictsHorizonsOnSyncTipError` | retained as `TestApplyScanBatchRollsBackDerivedAddressCache`: it fails `SetSyncedTo` after extending a horizon and proves the live derived-address state rolls back |
| `TestDBPutTargetedBatchEvictsHorizonsOnTxError` | retained as `TestApplyScanBatchRollbackEvictsThroughAdvancedNextIndex` |
| `TestDBPutRewind_RollbackFailureRestoresTip` | retained as `TestRollbackToBlockAtomicRollsBackSyncTipOnFailure` for SQL and `TestRollbackToBlockAtomicRollsBackSyncTipOnRollbackError` for kvdb |
| `TestDBGetScanData_MultipleTargets` | retained as the restored `TestStoreScanHorizonsTargetedImportedNotResolvedAsDerived`, which supplies two distinct targets and requires both horizons |
| `TestDBPutTxns_InternalAddressAsChange` | retained as kvdb `TestApplyTxBatchResolvesCreditCandidates` (`kvdb/txstore_test.go:2203`), the closest analogue, with `TestApplyTxBatchRecordsTxAndSyncedTo` and the narrower `TestCreateTxUnminedWithCreditSuccess` covering the same property at the batch and Create boundaries. All three mock `ManagedAddress.Internal()` true and require `Credit.Change`; the kvdb adapter derives the flag in production at `kvdb/txstore.go:892` (`expectedChange = managedAddr.Internal()`). The same-named case in the shared DB suite asserts no change flag, and SQL has no credit-change column at base or head — a backend divergence owned by deferred query-surface work, not a lost property |

Two rules follow. A move or rename is a ledger entry, not an exemption: the gate
is path-qualified precisely so a relocation is visible. And a count is not a
disposition — the 37 `db_ops` names all appear in the ledger even though one
reason and one commit cover the group.

### 5.9 Falsifier rule

Protect properties, not historical test names. Every retained production fix
must keep one effective test at the narrowest useful boundary. For every
renamed or consolidated falsifier, update `opus-report.md` with:

- the retained property;
- the final test/subtest;
- the mutation or reverted hunk;
- the observed failure.

Delete obsolete report rows for reverted or deferred production behavior. Do
not retain a duplicate test solely because a historical report named it.

## 6. Correct stale tests and comments

### 6.1 Removed live-watch expectations

Correct all three stale sync tests, not only `TestInitChainSync`:

- `TestInitChainSync`;
- `TestRunSyncStep_Success`;
- `TestRunSyncStep_BroadcastError`.

Remove `expectEmptyScanData`, `NotifyReceived`, and post-catch-up live-watch
comments from these tests. Delete `expectEmptyScanData` when unused.

`TestInitChainSync` must use strict expectations for `IsCurrent`, the synced-tip
Store read, and `NotifyBlocks`, then assert both Store and chain mocks. Do not
rely on a `.Maybe()` helper for the synced-tip read. The two `runSyncStep` tests
must also assert their actual Store and chain expectations.

### 6.2 Harness failed-Create test

Make `TestTeardownWalletsClosesManagerAfterFailedCreate` deliberately fail
validation with a minimal invalid Wallet config:

- assert the returned Wallet is nil;
- assert `errors.Is(err, wallet.ErrMissingParam)`;
- run centralized teardown;
- prove the database can be reopened afterward;
- bound any lock wait so a missing Manager Close fails rather than hangs.

Do not claim a short seed caused the failure and do not create a mock Chain only
to reach a later failure stage. The test subject is harness ownership.

### 6.3 Other stale text

Correct only stack-added or stack-rewritten text:

- the stale `wallet restart` reference needs no separate correction:
  section 4.5 deletes `TestManagerStopClosesNoStore`, which carried it;
- remove the claim that locked-wallet shutdown skips Vault `Lock`;
- correct the script-only import comment that says spendable wallets accept it;
- remove stale runtime-factory, close-owner, and accepted-response wording;
- remove review chronology and step-by-step narration from touched tests.

## 7. PSBT completion record

There is no open owner question and no Task-1000 work.

- Amend Task 315 DoD 6 to require existing mock-backed PSBT routing, including
  a non-zero account row.
- Explicitly mark real-backend address-driven PSBT signing as not delivered and
  deferred because of the pinned-base coin-type defect.
- Remove the explanatory NOTE from `TestSigningAcrossBackends`.
- Record the exception in the new Task-316 section of `opus-report.md` and
  reference the existing BWR-010 entry without editing Task 1000.
- Do not create a new task or fix the PSBT behavior.

## 8. Commit ownership and history shape

Rewrite behavior-owning commits, not files. A file touched by several commits
does not have one owner.

| rev-6 commit | Task-316 edits folded into this commit |
| --- | --- |
| `0191cb119` | narrow account-secret query, accessor, adapters, and tests |
| `173447222` | wallet-secret genesis test consolidation |
| `da4faf2c0` | address-secret provenance and descendant accessor cleanup |
| `c60f40a3` | keyvault passphrase test consolidation only |
| `d2283ab37` | minimal kvdb Vault/address-secret bridge and tests |
| `a69152651` | signer/PSBT routing and non-duplicate regressions |
| `58d8d9754` | serial Wallet lifecycle-to-Vault routing only |
| `23f81241f` | retain the kvdb lifecycle bridge; trim only its added comments |
| `2054150a2` | birthday export only; no unrelated cleanup |
| `899a13196` | Store sync routing with base targeted identity |
| `b6221ff78` | Manager-owned runtime Store and descendant conflict cleanup |
| `70adc46fa` | backend selection and centralized teardown only |

Fold a removal into the commit that introduced the removed behavior or test. If
a descendant edited that hunk, remove the descendant edit during rebase. The
final history must never introduce a deferred feature or duplicate and remove
it in a later commit.

Keep the 7/4/1 PR shape unless reduction makes a commit empty. Drop an empty
commit rather than preserving a no-op. Add no Task-316, fixup, review, cleanup,
or comment-only commit. Every rewritten commit must compile and test against its
parent and must be GPG-signed.

Commit messages keep the repository's subsystem prefix and present-tense
subject, at most 50 characters. Bodies explain the retained invariant, wrap
near 72 columns, and omit review chronology, temporary OIDs, and revision
numbers.

## 9. Execution protocol

### 9.1 Preflight and backups

Before rewriting:

1. Fetch `upstream` -- the remote that backs the PR heads, confirmed by
   `headRepositoryOwner = btcsuite` -- and every other remote that carries
   these three branches, without changing local branches. `origin`
   (`yyforyongyu/btcwallet`) is a mirror, not a PR head.
2. Verify the pinned base, all three local input heads, base -> #1272 -> #1291
   -> #1292 ancestry, and one lease pair per (remote, branch) rather than three
   OIDs overall. Today that is six pairs: `upstream` and `origin` each hold
   `impl-secret-store` = `6eae9f519082053c1dfcffd8e0c326c16d161ddf`,
   `impl-runtime-store-6` = `61d2e60c10467e1023ddb05acf98b3dac686e969`, and
   `wallet-default-sqlite` = `4d672cbcf3a9d162056198f956e7304af9e66c7a`.
3. Require only Task-316 candidate, validation, and branch-promotion worktrees
   to have no unrelated tracked changes. Record and ignore unrelated dirty
   worktrees. Untracked `.lgtm` files in the main worktree are expected; run
   mutating checks in dedicated clean worktrees under `.worktrees/`.
4. Create and verify these backup refs:

   ```text
   refs/backup/t316-pre/1272 = 58d8d97549a6900926ea32bb6af05865971a5cbb
   refs/backup/t316-pre/1291 = b6221ff78e156f10f2c8861a76a7dbddd891db61
   refs/backup/t316-pre/1292 = 70adc46faa8e275a56400a8c5bb09b4801d0c38d
   ```

5. Preserve failed candidate refs for diagnosis.

Initialize these candidate branches at the three input heads, then rewrite
them without moving the delivered branches:

```text
refs/heads/t316/candidate-1272
refs/heads/t316/candidate-1291
refs/heads/t316/candidate-1292
```

Abort if any input branch, parent, remote lease, signature, or Task-owned
worktree differs from the recorded preflight.

### 9.2 Rewrite order

1. Rewrite #1272 chronologically and validate its boundary.
2. Rebase #1291 onto the rewritten #1272 head.
3. Rewrite #1291's four own commits chronologically and validate its boundary.
4. Rebase #1292 onto the rewritten #1291 head.
5. Rewrite #1292 and validate the final boundary.
6. Validate every candidate commit and all three candidate PR ranges.
7. Promote candidates #1272 -> #1291 -> #1292 only after all local gates pass,
   with a compare-and-swap on every ref. `git reset --keep` protects a working
   tree but has **no** expected-old-ref guard, so it silently overwrites a
   delivered branch that advanced after preflight; do not use it to move a
   delivered ref. Require the owning worktree to be clean, detach it, and then
   promote with

   ```text
   git update-ref refs/heads/<branch> <candidate-oid> <recorded-rev6-oid>
   ```

   Promotion is phase-aware for the same reason section 13's recovery is: on a
   retry after a partial promotion, a ref already equal to its candidate is
   **skipped**, a ref still equal to its recorded rev-6 OID is promoted, and any
   third value aborts. Abort on a failed compare-and-swap; never move a
   delivered branch by reset or unchecked force.

Do not use a delivered branch as scratch state. Do not run
`scripts/check-each-commit.sh`: it rebases and therefore validates different
OIDs. Check each exact candidate OID non-mutatingly in a detached validation
worktree as section 10.1 requires.

## 10. Verification

### 10.1 Per rewritten commit

Task 315 DoD 7's test standard remains binding. Enumerate every exact candidate
OID in each PR range. In a detached validation worktree, check out each OID and
run `scripts/check-commit.sh` without rebasing. For each exact OID, also:

- run the narrow tests for the owned package and behavior;
- run `go vet ./...`;
- run `git diff --check <parent> <commit>`;
- verify the transaction-boundary and prohibited-symbol gates applicable at
  that point;
- verify generated sources are unchanged relative to the rev-6 owner commit;
- verify the subject/body limits and `%G? = G` for that exact OID;
- re-read the body of every commit whose patch this task changes and require it
  to describe what the commit now contains. Limits are not truthfulness: when
  section 4.1 strips the store-identity model out of `899a13196`, its body must
  lose the store-identity description, and a commit that loses a test must stop
  describing it.

From `899a13196`'s replacement onward, assert zero references to the removed
target-identity symbols and zero targeted-selector comments treating
`AccountScope.Account` as a Store ID. Untargeted full-scan state may retain the
private use authorized by section 4.2. At every applicable OID, run the exact
resolver test and source review defined in section 4.1. From each behavior's
introducing commit onward, assert the deferred test/helper names are absent.

### 10.2 Per PR head

Run Task 315 DoD 8 from a pristine worktree at each applicable head. This
includes the coupled formatting/generation checks; plain `gofmt` is not a gate:

```text
make sample-conf-check
make fmt-check
make tidy-module-check
make rpc-check
make sqlc-check
make sql-format-check
make sql-lint-check
go install -v ./...
make protolint
make lint-check
make unit
make unit-cover
make unit-race
go test -count=1 ./...
go test -run '^$' -tags='itest dev nolog' ./itest
```

Run the **existing** SQLite and PostgreSQL DB integration gates (`itest-db`,
`itest-db-race`) where Task 315 assigns them, unchanged. CI runs both over the
`[sqlite, postgres]` matrix as a required job, and this stack adds compile-forced
pg Store methods the shared suite exercises, so a local run is the parity gate
that catches compile-forced breakage before a push. It verifies only the already
shared Store contract: add no PostgreSQL case, row, harness, Manager path, or
completion criterion (amendment 4). After every mutating check, require a clean
tracked status.

Also run the **base-test inventory gate** at each head. Section 3 rule 1 states
"preserve an unchanged base test" as a principle, but nothing enumerated base
deletions -- which is exactly why rev 6's removal of
`TestStoreScanHorizonsTargetedImportedNotResolvedAsDerived` survived three
review rounds. Enumerate mechanically, and **qualify by file before comparing**:
`git grep -h` would strip the path, so a deletion in one file is hidden by an
identical declaration in another. That is not hypothetical -- the pinned base
declares `TestConfigValidateErrors` in both
`wallet/internal/db/pg/config_test.go:45` and
`wallet/internal/db/sqlite/config_test.go:45`. Resolving paths after `comm`
cannot recover an entry `comm` already collapsed.

This block is normative: run the complete here-document **under Bash** — it
relies on Bash semantics. First bind `candidate_oid` in the caller to the exact
validated full OID under test, never a branch name. The leading parameter check
fails before Bash starts if the caller did not bind it. Materialize both
inventories as files, pin the collation for the whole block, and cross-check the
result with a second, collation-independent method.

```bash
: "${candidate_oid:?bind candidate_oid to the exact candidate OID}"
HEAD_OID="$candidate_oid" bash <<'INVENTORY_GATE'
set -o pipefail
set -u
export LC_ALL=C

# Bind every input before use: an unset $dir would resolve the outputs to
# /base.txt and friends.
base=2ea76dfc5e85acf84014a21e4813cb23945250fa
# Supplied by the caller: the exact validated candidate OID, never a branch
# name, for the same reason section 12.2 pushes an OID. set -u fails here if it
# is unset, so there is no unbound-variable path into the gate.
head="$HEAD_OID"
dir="$(mktemp -d)" || exit 1
trap 'rm -rf "$dir"' EXIT

# --threads=1 is load-bearing, not a style choice: see below.
list_tests() {  # $1 = revision, $2 = output file
    git -c grep.threads=1 grep -n -E '^func Test[A-Za-z0-9_]*\(' "$1" -- \
        ':(top,glob)**/*_test.go' |
        awk -F: '{
            name = $4
            sub(/^func /, "", name)
            sub(/\(.*/, "", name)
            print $2 "::" name
        }' |
        sort -u > "$2" || return 1   # pipefail alone is not enough: without
                                     # this the function returns test -s below,
                                     # so a partial-but-nonempty run passes
    test -s "$2"        # an empty inventory is a gate failure, never success
}

# Generate each inventory twice and require the pair to match, so any residual
# nondeterminism fails the gate instead of silently shrinking it.
for rev_out in "$base:base" "$head:head"; do
    rev="${rev_out%%:*}"; out="${rev_out##*:}"
    list_tests "$rev" "$dir/$out.txt"   || exit 1
    list_tests "$rev" "$dir/$out.2.txt" || exit 1
    cmp "$dir/$out.txt" "$dir/$out.2.txt" || exit 1
done

comm -23 "$dir/base.txt" "$dir/head.txt" > "$dir/deleted.txt" || exit 1

# Validates the set operation after enumeration -- not the completeness of the
# enumeration itself, which only the serial generator and the cmp above assure.
grep -F -x -v -f "$dir/head.txt" "$dir/base.txt" > "$dir/deleted.check"
grep_status=$?                   # not `status`: zsh reserves that name, and
                                 # this workspace's default shell is zsh
test "$grep_status" -le 1 || exit 1   # 1 means "selected nothing", i.e. zero
                                      # deletions -- legitimate; 2+ is an error
cmp "$dir/deleted.txt" "$dir/deleted.check" || exit 1

# The deleted inventory IS the gate's output: without this line the result
# strands in the scratch directory and the block "emits" nothing.
cat "$dir/deleted.txt" || exit 1
INVENTORY_GATE
```

Three defects this form closes, each reproduced in this workspace on git 2.55.0:

- **a nondeterministic generator.** Default multi-threaded `git grep` silently
  drops matches at a *fixed* revision and still exits 0. Five runs of the
  declaration grep at the pinned base returned 1353, 1353, 1353, 1351, 1352
  lines; `git -c grep.threads=1` returned **1355** on three consecutive runs,
  and 1412 twice at the rev-6 head. Every threaded sample was short, never
  long, so the threaded runs were losing declarations. Writing that output to a
  file cannot recover them, and comparing two corrupted files by two methods is
  one witness, not two -- which is why the generator is pinned and each
  inventory is generated twice and `cmp`-ed.
- **exit status.** Process substitution does not propagate a failed `git grep`,
  so a bad revision would surface as a clean inventory. The checked calls and
  the `test -s` guard make that a gate failure.
- **collation.** `comm` compares under the ambient locale, so pinning `LC_ALL=C`
  on `sort` alone lets a bare `comm` mis-merge C-sorted input, silently and with
  exit 0 -- one observed run inflated the inventory to 92 entries. The
  mis-merge direction is undefined, so the same mechanism can *hide* a real
  deletion. Every text tool in the pipeline runs under the same `LC_ALL=C`.

With the generator pinned, the gate is reproducible. Against the rev-6 inputs it
emits **11** entries at #1272 and **51** at #1291/#1292; against the rewritten
candidates it must emit **7** and **43**, the difference being section 5.8's
eight restorations. `comm` and `grep -F -x -v -f` agree at every head.

Use a Task-owned temporary directory for `$dir` and commit no artifact.

Run it base-to-each-PR-head and base-to-final. A restored declaration is
discharged by its **absence** from the output -- it exists again at the same
`path::TestName`, so the gate cannot emit it. Every line the gate *does* emit
must map to exactly one non-restored ledger entry: minimally adapted or renamed
(with the new name listed), replaced by a named retained test, or removed
together with its production path and a section-5 citation. A same-named test in
another file does **not** discharge an entry. Any unmapped line fails the gate.

### 10.3 Final #1292 head

Run exactly one existing Manager case on each backend:

```text
make itest chain=btcd db=kvdb icase=manager nocache=1
make itest chain=btcd db=sqlite icase=manager nocache=1
```

Each invocation must execute only `manager_create_wallet`, and that case must
contain no restart/unlock scenario added by this stack.

Also require:

- the Task-315 public Wallet behavior matrix **as reduced here** — in
  particular its lifecycle row is now: vault lock/unlock on both backends, plus
  the serial Stop-time vault lock proved by the adapted base
  `TestControllerStop` (unlocked wallet, exactly one `Vault.Lock`, stopped
  state). Cache retention is owned by base cached-Load identity and the reduced
  SQLite cached-Load case, not by a Manager Stop test -- there is none. No
  restart scenario, no Unlock -> Stop -> Start -> Unlock sequence, and no
  broader Start/Stop race are required at any level;
- focused SQL recovery tests with unequal account number and Store ID;
- the relevant package results within the per-head `make unit-race` gate;
- the transaction gate at 18 -> 0 -> 0 for #1272/#1291/#1292;
- all final commits linear and GPG-valid.

## 11. Metrics and scope ledger

The rev-6 final baseline is:

| metric | rev-6 value |
| --- | ---: |
| all-file A+D | 16,645 |
| changed files | 106 |
| production-Go A+D | 5,865 |
| test-only A+D | 10,636 |

Use `opus-report.md` section 7.6b's exact all-file and production-Go predicates.
Add these exact predicates:

```bash
# test-only A+D
git diff --numstat <base>..<head> -- ':(top,glob)**/*_test.go'

# changed-file count
git diff --name-only <base>..<head>
```

Sum additions and deletions using the same method as section 7.6b and count the
second command's output. Record cumulative base-to-head and disjoint
base-to-#1272, #1272-to-#1291, and #1291-to-#1292 rows.

The final candidate must:

- remain below Task 315's old 22,469 all-file and 9,162 production-Go gates;
- strictly reduce all-file, production-Go, and test-only A+D from rev 6;
- not increase the 106 changed-file count;
- map every changed production file to a retained Task-315 requirement or
  compile-forced shared interface implementation;
- include one disjoint removal/consolidation ledger, without adding overlapping
  reviewer estimates.

A non-decreasing all-file, production-Go, or test-only A+D **fails execution**.
It is a gate, not a note: identify the exact retained behavior that offsets the
reduction, then stop and proceed only if the owner explicitly approves that
named behavior and that exact metric exception, as Task 315 DoD 10 requires.
The changed-file count remains non-increasing as written above. A lower total
never authorizes an unmapped production file.

## 12. Records, push, and external state

### 12.1 Local candidate records

After code and tests settle, but before any push:

- append a Task-316 correction section to `opus-report.md`; do not rewrite
  historical rev narratives;
- append a dominant Task-316 status and old-rev-6 to final-OID map to the
  commit manifest. Keep the old 43-commit dispositions as historical evidence,
  but explicitly supersede old `pushed`, `complete`, and `nothing remains`
  claims;
- update Task 315, Task 316, and `INDEX.md` with full candidate OIDs, metrics,
  the PSBT exception, and `awaiting owner-authorized push` status;
- update falsifier rows to their retained final tests;
- leave Tasks 1000, 10001, and 10002 untouched.

Do not describe the candidate as remote-final or CI-validated.

### 12.2 External mutation requires authorization

This task does not authorize any push, PR-body edit, review reply, or thread
resolution.

**Authorization is per operation class, and nothing is inferred.** A push
authorization is not permission to edit PR metadata, and neither authorizes the
compensating force push section 13 would perform. Before any external work,
require the owner to name separately:

1. every push tuple `(remote, ref, old OID, new OID)`. Name each remote
   explicitly -- do not infer `upstream` from it backing the PR heads, and do
   not infer `origin` from its tracking refs matching;
2. either advance permission for the exact compensating rollback tuples, or an
   instruction to stop and ask after a partial push;
3. PR-body edits, by PR;
4. review replies and thread resolutions, at the exact candidate heads.

With those in hand:

1. Re-fetch and verify every authorized remote ref and all validated local OIDs.
2. Work one branch at a time in #1272, #1291, #1292 order. For each branch: push
   the first authorized remote, verify that ref alone, then push each remaining
   authorized remote and verify it; only once every authorized remote holds the
   candidate, verify the PR `headRefOid`. Do not require an unpushed remote to
   equal the candidate in between.
3. **Push the exact validated OID, never a mutable local branch.** A destination
   lease stops a remote that moved; it does nothing about a local branch that
   moved after validation, which would satisfy the lease and publish an
   unvalidated tip. Use a full-OID source refspec together with the
   branch-and-remote-specific lease from section 9.1:

   ```text
   git push --force-with-lease=<ref>:<remote-old-full-oid> <remote> \
       <candidate-full-oid>:<ref>
   ```

   Never use plain `--force`.
4. Require CI green at those exact OIDs.
5. Immediately before **every** metadata mutation -- each PR-body edit, each
   reply, each thread resolution, and the completion transition -- re-fetch all
   authorized remote refs and the affected PR head and require them to equal the
   validated candidates. A candidate can pass exact-OID CI and be replaced
   before the next call.
6. Snapshot each old PR body and record the exact body this task writes, then
   update all three bodies to describe only the final retained design.
7. Re-fetch every review-thread page at the pushed OIDs. Record thread ID, URL,
   PR, path, disposition, replacement OID, and reply. Re-fetch afterwards and
   require zero unresolved threads.
8. Only then mark Tasks 315 and 316 complete and reclassify the review files as
   historical evidence. Never mark them complete after a partial metadata sweep.

Do not trust rev-5 CI, a pre-push thread snapshot, or a hard-coded thread count.

## 13. Rollback

For a local rewrite failure, recovery is **phase-aware**. Section 9.2 rewrites
and validates candidates without moving a delivered branch, so before promotion
there is nothing to restore, and a blanket
`git update-ref <ref> <backup-oid> <candidate-oid>` would fail its
compare-and-swap on every ref that still holds its rev-6 OID. In all phases:
stop, preserve the failed candidate refs, and verify no Task-owned worktree
changed.

**Before promotion** -- no delivered ref has moved:

- verify each delivered ref still equals its recorded rev-6 input OID;
- move nothing. A ref that does not match is an unexplained external change:
  stop and report rather than writing over it.

**During or after promotion** -- some refs may have moved:

- inspect the delivered refs in **reverse** dependency order, #1292 -> #1291 ->
  #1272, in their verified clean, detached owning worktrees;
- a ref equal to its candidate is restored with
  `git update-ref refs/heads/<branch> <backup-oid> <candidate-oid>`;
- a ref equal to its recorded rev-6 OID was never promoted: skip it;
- any third value aborts the whole rollback.

Never `git reset --keep` a delivered ref: it has no expected-old guard.

For a partial remote push:

- stop before pushing another branch or another remote;
- restore every remote that already changed, on every branch it changed, in
  reverse dependency order, each with a lease whose expected OID is the
  candidate currently on **that** remote -- not a single shared expectation --
  and with the captured old OID as an explicit push **source**, never a local
  branch name:

  ```text
  git push --force-with-lease=<ref>:<candidate-full-oid> <remote> \
      <remote-old-full-oid>:<ref>
  ```

- a compensating force push is itself an external mutation: perform it only
  under the advance rollback grant section 12.2 requires, otherwise stop and
  ask;
- this covers a failure between the first and second remote for one branch:
  that branch is restored on the remote that took it while the remote that did
  not is left alone;
- verify each restored ref, then each PR `headRefOid`.

For a partial metadata failure -- some PR bodies or review threads changed:

- re-fetch before doing anything else; restore a body **only** if its current
  value still equals the exact text this task wrote, so a concurrent human edit
  is never overwritten;
- treat replies and thread resolutions as irreversible: record precisely what
  completed, make no further mutation, and stop for owner review;
- do not mark Tasks 315 or 316 complete.

For post-push CI failure:

- make no PR-body or review-thread mutations;
- either repair and rerun all gates, or restore rev 5 only on explicit owner
  instruction; either way lease per remote against the OID that remote actually
  holds, and re-verify every remote after the operation;
- if PR bodies or threads were somehow already changed, do **not** restore a
  snapshot unconditionally -- apply the partial-metadata rule above, including
  its equality guard, so a concurrent human edit is never overwritten.

Keep all backup refs through acceptance.

## 14. Definition of done

1. Task authority is consistent: Task 316 is the sole active reduction plan and
   Task 315 is the frozen specification amended only by section 1.
2. The targeted path uses base's name-preferred Store lookup and
   account-number semantics, with the name resolved from one Store snapshot
   instead of a walletdb transaction. Imported rows are indexed by `AccountID`
   **only** behind `s.addrStore != nil`; no SQL path interprets `AccountID` as
   `waddrmgr.AccountScope.Account`, and a SQL numeric target matching no derived
   row returns `db.ErrAccountNotFound`. Removed identity symbols and the two
   deleted structural tests are absent from every applicable final commit,
   `TestStoreScanHorizonsTargetedImportedNotResolvedAsDerived` passes on the
   real kvdb backend, and section 4.1's exact-OID source review plus repurposed
   resolver test prove the SQL boundary.
3. SQL full-scan derivation and horizon Store IDs remain correct with unequal
   account number/ID and numeric-collision proof, and Store ID never becomes
   `DerivedAccountNumber`.
4. The account-secret accessor returns only cloned encrypted private-key
   material; Close-race support and unused public metadata decryption are gone.
4b. Manager coverage asserts caller-visible behavior only: **no test references
   or asserts** `m.kvdbBound`, `m.cached`, `w.store`, `w.addrStore`,
   `w.txStore`, `m.chainParams`, or `w.cfg.*`, reads through a private Store
   handle, or calls `openKVDBBinding`/`buildWallet` directly (amendment 5).
   This governs tests only -- the production `m.kvdbBound` field still owns the
   legacy binding and stays.
5. Deferred watch-only import, secret-witness, lifecycle-race, and fabricated
   import tests and their production-only support are gone. The minimized
   watch-only Vault `Lock` no-op regression remains.
6. Every exact consolidation in section 5 is complete; base tests and protected
   ownership boundaries remain; section 5.8's base-test deletion ledger is
   discharged entry by entry against the **rev-6 input audit** of 11/51/51 —
   all 37 `db_ops` names enumerated, the seven property-outlives-wrapper
   exceptions mapped individually, every rename listed — and section 10.2's
   gate, run with the serialized generator and its double-generate `cmp`,
   emits **7** entries at the #1272 candidate and **43** at the #1291 and #1292
   candidates, every one mapped to a non-restored ledger entry and none
   unmapped.
7. The Manager itest has one original create/start/sync scenario and no added
   restart lifecycle.
8. The three stale live-watch tests assert current calls, the harness failure
   test proves its stated failure and cleanup, and stale comments are corrected.
9. Mock PSBT account routing remains covered; real-backend address-driven PSBT
   is recorded as deferred and no PSBT production behavior changes.
10. Every retained production fix has one effective final falsifier and the
    report maps renamed tests to their mutation failures.
11. Every rewritten commit passes section 10.1 and is linear, atomic, and
    GPG-valid. Every PR head passes section 10.2.
12. Both final Manager commands pass exactly one unchanged scenario, and all
    final-head focused and race gates pass.
13. Final metrics and a disjoint de-duplication ledger satisfy section 11. No
    generated or unrelated production scope was added.
14. Candidate records use full final OIDs and remain marked unpushed until the
    owner authorizes section 12.2.
15. After authorization, exact-OID CI and paginated thread gates pass before
    Tasks 315 and 316 are marked complete.

## 15. Evidence

### 15.1 Plan-review dispositions (2026-08-01, second round)

| finding | disposition |
| --- | --- |
| codex 1 — bootstrap already complete, conditional status blocks a literal executor | **taken.** Status is now "active — sole executable reduction plan"; §1 marks the bootstrap satisfied and keeps the amendments binding; §12 keeps the final-OID work |
| codex 2 — PostgreSQL execution gates contradict §4.6 | **taken in scope, not in verification.** §1 gains amendment 4 (compile fallout only; no pg Manager, harness, daemon, matrix, row, or completion gate) and §5.6 is reworded to protect shared *source*. The existing `itest-db`/`itest-db-race` pg run is **kept**: `.github/workflows/main.yml:170-196` runs it over `[sqlite, postgres]` as a required job today, and this stack adds compile-forced pg Store methods the shared suite exercises, so deleting the local run would not remove the gate — it would only move discovery to CI. Codex's own item 4 concedes hygiene for compile-forced code |
| codex 3 — 315 §5.5 and DoD 6 still require the removed restart sequence | **taken.** 315 §5.5 keeps the forced `keyVault.Lock()` ruling but its test prescription is marked amended; DoD 6's lifecycle row now names the adapted base `TestControllerStop` (exactly one `Vault.Lock`) plus `TestManagerStopClosesNoStore`, and forbids the sequence at any level. §10.3 names that reduced matrix directly instead of pointing at the stale table |
| codex 4 — refresh the INDEX summary | **taken.** Four amendments, the exact test-correction scope (three stale sync tests plus the harness failed-Create test), the PostgreSQL deferral, and review-the-diffs made conditional on §9/§10 |
| fable 1 — sparing `TestHandleUnlockReqRepeatDoesNotLock` | **premise not confirmed** (and retracted by the reviewer in round 3). The claim was that the rebuilt handler "deliberately implements no-lock-on-repeat-unlock, with an in-code comment". It does not: `handleUnlockReq` differs from base only by routing `DBUnlock` to `keyVault.Unlock`, its sole added comment is about auto-lock ownership, and the "already unlocked returns `ErrVaultUnlocked` without validating" contract plus `TestWalletVaultUnlockAlreadyUnlockedReturnsError` are both at the pinned base (`74fefd25d` is an ancestor of `2ea76dfc5`). The interim two-row fold this round proposed was itself removed in round 3 — see §15.2 |
| fable 2 — push both remotes | **taken, then refined — superseded by §15.3.** Historical wording: "§12.2 pushes upstream then origin with per-remote leases." That is no longer the operative rule; §12.2 and §13 as written now govern, and matching tracking refs are not authorization to update a mirror |

### 15.2 Plan-review dispositions (2026-08-01, third round)

| finding | disposition |
| --- | --- |
| codex 1 — `TestManagerStopClosesNoStore` still runs the restart under private helpers | **taken, and it was a real miss.** §4.5 and §5.4 previously said only to correct the test's comments; the rev-6 body still does start, unlock, Stop, `startLoadedWalletForTest` again, unlock again, `NewAccount("post-restart")` — verified in `wallet/manager_test.go`. Both sections now specify a reduced body. **Partly superseded by §15.4:** the "one direct Store read through the Manager-owned handle" clause this round added was itself a private-structure assertion and is removed under amendment 5; the reduced body is one start/Stop plus the cached-`Load` assertion, with the adapted base `TestControllerStop` as the sole Stop-to-Vault dispatch proof |
| codex 2 — the two replacement rows preserve the duplicate under a new name | **taken; supersedes the round-2 fold.** Every Unlock error takes one base error-return branch already proved by base `TestHandleUnlockReq_Errors/DBUnlock_Failure` (already minimally adapted to expect the Vault call), and "wrong passphrase while already unlocked" is unreachable through the real `WalletVault`. §5.1 now deletes the test with no replacement row, and the round-2 row above is marked superseded |
| codex 3 — the two-remote protocol is not atomic and infers mirror authorization | **taken.** `upstream` backs the PR heads (`headRepositoryOwner = btcsuite`, checked); `origin` is the `yyforyongyu` mirror and is updated only when the authorization names it — matching tracking refs are not authorization. §9.1 records one lease pair per (remote, branch), six today; §12.2 pushes per branch as upstream → verify → each authorized mirror → verify, and only then checks `headRefOid`; §13 restores every remote that changed, per-remote lease, covering a failure between the two remotes of one branch |
| fable residual — §13 said "restore each already-pushed `upstream` branch" | **taken**, in the same §13 rewrite as codex 3 |
| codex 4 — three documentation nits | **taken.** §2 names the exact inventory (three stale sync tests plus the harness failed-Create test); amendment 4 now forbids a *new* acceptance criterion, case, row, or matrix while keeping the existing CI compatibility matrix mandatory; the over-long Task 315 DoD 8 line is rewrapped |

### 15.3 Plan-review dispositions (2026-08-01, fourth round)

| finding | disposition |
| --- | --- |
| **blocker — targeted imported-xpub requirements are incompatible** | **taken; the most serious finding of the cycle, and it was mine to catch.** Verified in the tree: `db.GetAccountWithOps` rejects any number lookup resolving to an imported account **at the pinned base** (`accountstore_getaccount.go:85-90`), so §4.1's number-only rule could never load a targeted imported account; base resolves the name first and proves it with the real-backend `TestStoreScanHorizonsTargetedImportedNotResolvedAsDerived` (`2ea76dfc5:wallet/syncer_test.go:2367`); **rev 6 deleted that base test** and defended the loss with a `scanSelectorKey` comment asserting base could not target an imported account distinctly — which is false. §4.1 is rewritten: `resolveScanTargets`/`resolveScanTarget` stay in their base role of filling `AccountName`, sourced from one `Store.ListAccounts` snapshot instead of `walletdb.View` (kvdb's `AccountID` is the internal non-masked number, the same value base read via `scopedMgr.AccountName`); `scanTarget` returns to its base `{Scope, Account, AccountName}` shape; the base test and its `createImportedXpubAccount` fixture are restored; DoD 2 and §5.2 updated to match. **Partly superseded by §15.4:** only the plural `resolveScanTargets` is a base function — the singular is rev-6's per-target matcher and is removed — and the imported `AccountID` match is kvdb-only |
| codex 4.1 — Task 315 DoD 11 still prescribes the superseded push/rollback order | **taken.** DoD 11 now states the invariant only and defers the procedure to §316-9.1/12.2/13, retaining both backup generations (`refs/backup/t315-pre/*` for the pre-315 heads, `refs/backup/t316-pre/*` for the rev-6 inputs) |
| codex 4.2 — the historical Fable push row reads as present-tense authority | **taken.** The §15.1 row is marked superseded and points at §15.3 |

### 15.4 Plan-review dispositions (2026-08-01, fifth and sixth rounds)

The fifth-round codex review and the sixth-round updated gpt review (now a
review of Task 316, not only the rev-6 branches) found the same backend leak
independently; codex's 11:39 assessment of that gpt file narrowed two fixes.

| finding | disposition |
| --- | --- |
| **blocker — SQL treats `AccountID` as a targeted selector** | **taken; my error in the fourth-round §4.1 rewrite.** I justified the imported fallback with kvdb semantics but wrote it unconditionally. SQL sets `AccountID` from `accounts.id` (`data_types.go:380-395`; the migrations keep `account_number` separate), so a target naming no derived account could select whichever imported row shared that integer — Store-ID selector behavior by accident. §4.1 now indexes derived rows by `AccountNumber` on every backend, imported rows by `AccountID` **only** behind `s.addrStore != nil` (the existing boundary at `syncer.go:336`), returns `db.ErrAccountNotFound` on SQL, lets a derived mapping win any collision, keys the index on `waddrmgr.AccountScope` instead of retaining `scanSelectorKey`, and adds a source gate plus DoD 2 language. Per codex's narrowing, the proof is **one** repurposed mock test — the `TestResolveScanTargetsStoreOnlyAccount` footprint, renamed — not a new real-backend SQL suite |
| **`resolveScanTarget` (singular) is not a base function** | **taken; my imprecision.** Only the plural exists at base — the "6 hits" a reviewer cited are substring matches. §4.1 now keeps the plural in its base role and removes the singular with the rest of the rev-6 identity layer; the §10.1 symbol gate lists it |
| gpt 2 / codex 2 — the push source is not pinned | **taken.** A destination lease says nothing about a local branch that moved after validation. §12.2 and §13 now require a full-OID source refspec alongside the lease, both for the push and for the compensating rollback |
| gpt 3 / codex 3 — one vague authorization covers every external effect | **taken.** §12.2 now requires authorization to enumerate push tuples, the compensating-rollback grant (or an instruction to stop and ask), PR-body edits, and review actions separately, with every remote named rather than inferred — including `upstream` |
| gpt 5 / codex 4 — `git reset --keep` has no compare-and-swap | **taken.** §9.2 and §13 now promote and restore delivered refs with `git update-ref <ref> <new> <expected-old>` in a clean detached worktree, aborting on a failed swap |
| gpt 6 / codex 5 — post-CI metadata mutations unguarded | **taken.** §12.2 re-verifies every authorized remote ref and PR head immediately before each body edit, reply, resolution, and the completion transition; §13 gains a partial-metadata recovery rule that restores a body only when it still equals the text this task wrote, treats replies and resolutions as irreversible, and forbids completion after a partial sweep |
| gpt 4 / codex 6 — Manager tests assert private structure | **taken as owner ruling; amendment 5.** This reverses two earlier dispositions of mine that were themselves reviewer-requested: §4.5's private Store read (codex round 3) and retaining `TestManagerBuildWalletFailureLeavesNoBinding` (codex round 1). Task 315 §5.1's invariant survives — it describes an effect, not an assertion mechanism, and the public form is stronger. `TestManagerStopClosesNoStore` reduces to start/Stop plus a cached `Load`; `TestManagerKVDBRejectsSecondName` requires both second-name operations to fail and the first name to stay usable; `TestManagerBuildWalletFailureLeavesNoBinding` and the `m.cached` assertion are deleted. Task 315 §5.1, §5.5, and DoD 6 restated to match |
| gpt 7 / codex 7 — the metric reduction is advisory | **taken.** §11 now fails execution on a non-decreasing all-file, production-Go, or test-only A+D, proceeding only on an owner-approved named exception per Task 315 DoD 10 |
| gpt 8 / codex 8 / fable round 5 — records, and no gate on deleted base tests | **taken.** Task 315's "(59 at base)" corrected to **23**, matching its own authoritative gate; `INDEX.md` narrowed to the reduction decisions §5 lists; and §10.2 gains the mechanical `comm -23` base-test inventory, resolved to `path::TestName`, one ledger entry each, no same-name discharge across files, no artifacts committed. This closes the class that let rev 6's base-test deletion survive three rounds |

### 15.5 Plan-review dispositions (2026-08-01, seventh round)

gpt (six findings) and codex (five) overlap almost entirely; fable reported
zero new findings and verified every §15.4 disposition in source.

| finding | disposition |
| --- | --- |
| **blocker — the base-test inventory discards file identity** | **taken; my error.** I adopted the name-only `comm` command: `git grep -h` strips the path before `sort -u`, so a deletion in one file is masked by an identical declaration in another — and the pinned base really does declare `TestConfigValidateErrors` in both `pg/config_test.go:45` and `sqlite/config_test.go:45`. Resolving paths *after* `comm` cannot recover a collapsed entry, which directly contradicted the next sentence in the same section. §10.2 now emits `path::TestName` before comparing, keeps the `:(top,glob)**/*_test.go` pathspec, and requires both `git grep` invocations to be checked because process substitution hides their exit status — an empty inventory must never read as success |
| **high — amendment 5 only partially applied** | **taken.** Two parts. (a) The reduced Stop test overclaimed: `Manager.Load` returns the cached pointer without touching the Store (`manager.go:564-569`), so it proves cache retention, not Store lifetime. It was renamed `TestManagerStopKeepsWalletCached` and every Store-lifetime claim struck from §4.5, §10.3, Task 315 §5.5, and DoD 6. **Superseded by §15.6:** with the Store-lifetime claim gone the test had no reachable falsifier, so it is deleted outright rather than renamed. (b) Four retained Manager tests still asserted private structure — `w.addrStore`/`w.txStore` (`runtime_store_test.go:234-235`), `w.addrStore` (`:520`), `m.chainParams`/`w.cfg.*` (`manager_test.go:952-976`), `w.store` (`runtime_store_test.go:105`); §5.4 now removes each, `TestManagerKVDBRejectsSecondName` proves "still usable" with one named public `ListAccounts`, and the rule is scoped to stack-added or materially rewritten bodies so unchanged base tests are not churned. DoD 4b reworded to ban *test references*, not the production `m.kvdbBound` field |
| medium — retention lists contradict the deletion | **taken.** §5.6's "pre-binding, post-binding, and post-commit" bullet outlived §5.4's deletion of the only post-binding assembly test. §5.6 now retains pre-binding and post-commit only, and §5.3's reason for deleting `TestManagerLoadClosesRuntimeStoreOnMetadataError` is corrected to the actual ruling — privately injected malformed assembly is out of scope — instead of claiming another retained test covers it |
| medium — renamed tests had no authoritative names | **taken.** Fixed names, used consistently across 315/316/DoD: `TestResolveScanTargetsDoesNotUseSQLAccountID`, `TestFullScanRecoverySeparatesStoreIDAndAccountNumber`, `TestManagerStopKeepsWalletCached`. Historical §15 rows keep the old names on purpose. **Partly superseded by §15.6:** the third test is deleted, so only the first two names remain live |
| medium — amendment count inconsistent | **taken.** §1's lead sentence and Task 315's status paragraph now say five and name the public-test amendment; `INDEX.md` already did. §15.3's plural-plus-singular resolver statement is marked superseded |
| medium — post-CI body rollback bypasses the concurrency guard | **taken.** §13's unconditional "restore captured PR-body snapshots" bullet is replaced by a pointer to the partial-metadata rule and its equality guard, so a concurrent human edit can never be overwritten |

### 15.6 Plan-review dispositions (2026-08-01, eighth round) and self-audit

codex's eighth round raised three items; fable endorsed all three and added a
fourth; gpt, read after the others landed, raised two more. The self-audit rows
are my own checks of the task files, not reviewer findings.

| finding | disposition |
| --- | --- |
| codex 1 / fable 2 — delete the reduced Manager Stop test | **taken.** Once the test stopped claiming Store lifetime nothing distinct was left: `Manager.Load` returns the cached pointer without touching the Store (`manager.go:564-569`) and `Wallet` holds no Manager back-reference, so `Wallet.Stop` cannot evict `Manager.wallets` — the assertion has no falsifier this stack can reach, and what remains composes base `TestControllerStop` with base `TestManagerLoad_ExistingWallet`. Deleted from §4.5, §5.4, §6.3, §10.3, and Task 315 §5.5 and DoD 6. This retires the rename introduced one round earlier |
| codex 2a / fable 3a — amendment 5 still ordered a Store-lifetime proof | **taken.** The clause was line-wrapped ("and Store\nlifetime"), which is why a single-line grep missed it. Struck; amendment 5 now states this stack carries **no** Manager-boundary Store-lifetime proof |
| codex 2b / fable 3b — the SQLite test's name and retained assertions contradict | **taken.** Renamed `TestManagerSQLiteCreateLoadCached`, keeping only public SQLite `Create` plus same-Manager cached `Load` identity; §5.3's "nil legacy-sidecar assertions" clause removed to match §5.4; §5.6's vague post-commit bullet replaced by the exact `TestManagerCreateUsesCommittedWalletRow` regression |
| codex 3 / fable 1 + 4 — the gate block is not fail-closed and mis-collates | **taken**, and the compare step hardened past both suggestions: files instead of process substitution, whole-block `LC_ALL=C`, `test -s`, and a second comparison method. **Diagnosis partly wrong, corrected in §15.7:** the 92-entry run was locale-`comm` garbage, but the 52-vs-53 spread was the *generator*, not collation, and the `txstore_pg_only_test.go` phantom came from there too. Two compare methods over the same corrupted files are one witness |
| **self-audit — §5 dispositioned only the tests reviewers named** | **new §5.8.** Running the gate now shows **53** base test declarations absent at the rev-6 head, of which **15 were named nowhere in Task 316**. §5.8 records the measured inventory with a per-file disposition: 37 in `wallet/db_ops_test.go` (removed with `wallet/db_ops.go` per Task 315 §1) collapse to one entry **except** seven whose behavior moved to the Store path rather than disappearing; `TestHandleLockReq_Errors` is a rename to `TestHandleLockReqErrors`; the base `TestDerivePubKey*` trio maps to the head's Store-path set; `TestWalletVaultEncryptSelectedRuntimeKeys` and `TestCreateWalletWithBirthday` are file **moves**, not deletions; `TestWalletVaultChangePassphraseLockedRequiresUnlockedState` is a deliberate behavior reversal under Task 315 §5.4. **Superseded by §15.7:** the count is 51 not 53, the two "move" rows were generator phantoms, and the six "no head equivalent" tests are now dispositioned exactly — one existing rename, one removed with its deleted production path, three restored unchanged, one adapted |
| **self-audit — `TestStampRecoveryAccountIDsCarriesStableID`** | **called out separately** because it sits inside §4.1/§4.2's own area: it is the base falsifier for stamping a durable Store ID onto recovery state, exactly the behavior §4.2 keeps and re-homes into the store-account registration. Rev 6 deleted it along with `stampRecoveryAccountIDs`, and no round noticed. §5.8 requires it restored against the retained stamping path |
| **self-audit — every `Test*` name in Task 316 verified against the tree** | **clean.** Cross-checked every backticked test identifier in the plan against base and rev-6 head: the only names present at neither are the three authoritative new names this task defines and the substring artifacts of `path::TestName`/`setupTestDB`/`createTestWalletWithMocks`. No phantom test names remain |
| gpt 1 — local rollback assumes every delivered ref was promoted | **taken.** §9.2 rewrites and validates candidates *without* moving a delivered branch, so before promotion every delivered ref still holds its rev-6 OID and the blanket `git update-ref <ref> <backup> <candidate>` would fail its compare-and-swap on all three — safe, but unable to complete. §13 is now phase-aware: before promotion, verify the refs and move nothing (a mismatch is an external change to report, not to overwrite); during or after promotion, walk them in **reverse** dependency order, restore only a ref equal to its candidate, skip one still at its rev-6 OID, and abort on any third value |
| **self-audit — promotion had the same phase assumption in the forward direction** | **fixed with it.** gpt named only the rollback, but §9.2 step 7's promotion CAS expects the recorded rev-6 OID, so a retry after a partial promotion would abort on refs already promoted. Promotion now skips a ref already equal to its candidate, promotes one still at its rev-6 OID, and aborts on any third value — the same three-way rule as recovery |
| gpt 2 — review-file roles are stale | **taken.** `INDEX.md` and §15.7 described gpt as only a rev-6 branch audit and codex as the sole Task-316 plan reviewer. Both files now carry several current plan-review rounds ahead of their older rev-6 sections, and both records say so |

### 15.7 Plan-review dispositions (2026-08-01, ninth round)

All three reviewers found the same defect one layer below the eighth round's
fix, independently and correctly. gpt was read after codex and fable; its
findings 1-5 converge with theirs and were already applied, and its 6 and 7 are
new.

| finding | disposition |
| --- | --- |
| **blocker — `git grep` enumeration is nondeterministic** | **taken; reproduced here before applying.** Default multi-threaded `git grep` silently drops matches at a *fixed* revision and exits 0: five runs of §10.2's declaration grep at the pinned base returned 1353, 1353, 1353, 1351, 1352, while `git -c grep.threads=1` returned **1355** three times and 1412 twice at the head. Every threaded sample was short, never long. The eighth round hardened the *compare* step and left the *generator* unpinned — and my "two compare methods" cross-check reads the same two files, so it is one witness, not two. §10.2 now pins `--threads=1`, generates each inventory **twice** and `cmp`s the pair, and states plainly that the set-comparison validates the set operation, not the completeness of enumeration |
| **the measured ledger was wrong** | **taken.** With the generator pinned the inventory is **51** at #1291/#1292 and **11** at #1272, `comm` and `grep -F -x -v -f` agreeing at every head. My two "file move" rows were phantoms: `TestWalletVaultEncryptSelectedRuntimeKeys` and `TestCreateWalletWithBirthday` exist at the **same** `path::TestName` in both revisions — nothing moved, and 53 = 51 + 2 dropped-then-misread lines. Both rows deleted; the five real file rows survive unchanged, which is itself evidence the rest of the audit held |
| codex 2 — the semantic ledger deferred decisions the code already answers | **taken, and I verified each before writing it.** The three `TestControllerChangePassphrase_Interrupted_Wait*` are **restored unchanged** (`sendReq`/`waitForResp` still present in rev-6 `controller.go`). `TestControllerStart_DBGetAllAccountsFail` is a **rename** to `TestControllerStartListAccountsFail`, already at the head. `TestResolvePrivKeyFallsBackAfterCacheMiss` is **not** restored — my claim that its fallback "still exists" was false: `resolvePrivKey`/`resolveDerivedPathPrivKey` went 2 → 0 in `signer.go`, and the surviving `DeriveFromKeyPathCache` hits are `deprecated.go` only. `TestStampRecoveryAccountIDsCarriesStableID` is an **unconditional** adaptation, no "or justify". **Partly superseded by §15.10:** the `TestControllerStart_DBGetAllAccountsFail` rename was later reversed — the base name is restored under rule 1; the fallback and stamp decisions stand |
| gpt 1-5 — same generator, count, phantom-move, and semantic-ledger findings | **converged.** gpt reproduced the instability independently (five runs of the unpinned pipeline returned 52, 52, 54, 61, 55; five serialized runs all returned 51) and reached the same 11/51/51 per-head counts, the same 37/7/4/2/1 distribution, and the same four semantic corrections. Already applied above; recorded here because three independent measurements agreeing is what makes 51 trustworthy |
| gpt 6 — the normative gate references unbound variables | **taken.** `$BASE`, `$HEAD`, and `$dir` were used without assignment, so an unset `$dir` would have written `/base.txt`. The block now sets `set -u`, binds `base` to the pinned base OID, binds `head` to the **exact candidate OID** rather than a branch name (same rule as §12.2's push source), and creates `dir` with a checked `mktemp -d` |
| gpt 7 — a partial inventory could still pass | **taken, and it was a real hole.** `list_tests` ended with `test -s "$2"`, so the function returned *that* status: `pipefail` caught a broken pipeline but nothing propagated it, and any partial-but-nonempty output passed. The pipeline now ends `|| return 1` before the emptiness check. `comm` is status-checked, and the cross-check `grep` accepts status 0 or 1 — 1 means "selected nothing", the legitimate zero-deletion case — and fails on 2+ |
| codex 3 — the 37-test `db_ops` row is not auditable | **taken, then corrected by §15.8.** This round added the names but initially grouped all 37 under one reason and one owning commit and incorrectly recorded `TestDBPutTxns_InternalAddressAsChange` as having no Store owner. Section 15.8 supersedes both claims: the final ledger is split 27+3+7 by owner, and the live kvdb change-marking falsifiers are named |

### 15.8 Plan-review dispositions (2026-08-01, tenth round)

codex and fable reviewed the ninth-round text and found the same three
accounting defects; gpt, read afterwards, reproduced two of them and added two
more. Every claim below was re-measured against the pinned trees before it was
written.

| finding | disposition |
| --- | --- |
| **blocker — 11/51/51 are input counts, not candidate expectations** | **taken.** The plan restores five declarations under their base `path::TestName`, so the candidate gate cannot still emit them: subtracting the three `TestControllerChangePassphrase_Interrupted_Wait*` (all three verified inside #1272's 11), `TestStoreScanHorizonsTargetedImportedNotResolvedAsDerived`, and `TestStampRecoveryAccountIDsCarriesStableID` gives **8/46/46**. DoD 6 required 11/51/51 at the *rewritten* heads, so a correct rebuild would have failed its own DoD. §5.8 now labels 11/51/51 the rev-6 **input audit**, §10.2 states the candidate expectation and says a restored entry is discharged by its **absence** from the output, and DoD 6 gates on 8/46/46 |
| **high — the promised 37-name ledger did not exist** | **taken.** §5.8 claimed the ledger "carries all 37 names" while only the seven exceptions appeared anywhere; `TestDBBirthdayBlock`, `TestDBPutBlocks`, `TestDBCreateWallet`, and `TestDBUnlock` had zero hits in any file. Section 5.8 now enumerates 27 ordinary #1291 file-deletion names, three separately owned #1272 Vault-routing names, and seven property-retention exceptions. The present-tense claim is now true of text that exists |
| **high — the final ledger row's evidence was false** | **taken; my error.** I recorded `TestDBPutTxns_InternalAddressAsChange` as having no owner after failing to find a change-marking assertion. The property is alive: the kvdb adapter derives it in production at `kvdb/txstore.go:892` (`expectedChange = managedAddr.Internal()`), and the final ledger names the closest analogue, `TestApplyTxBatchResolvesCreditCandidates`, plus `TestApplyTxBatchRecordsTxAndSyncedTo` and the narrower `TestCreateTxUnminedWithCreditSuccess`. All three mock `Internal()` true and require `Credit.Change`. SQL has no credit-change column at either revision, so the row records the SQL-side gap as a backend divergence owned by deferred work. The three remaining family citations are replaced by exact names (`TestApplyScanBatchRollsBackDerivedAddressCache`, the two `TestRollbackToBlockAtomic*` tests, and the restored two-target horizon test), all verified present at base and head |

| gpt 1 + 3 — input-vs-candidate counts, and the vague Store mappings | **converged** with codex/fable and already applied above. gpt independently derived the same 8/46/46 and the same six exact mappings |
| gpt 2 — the 37 `db_ops` tests do not share one owner | **taken; my grouping was wrong.** `TestDBUnlock`, `TestDBPutPassphrase`, and `TestDBPutPassphrase_Error` die at **#1272**, not with the #1291 file deletion: their subjects `DBUnlock` and `DBPutPassphrase` exist in base `db_ops.go` at :206 and :222 and are already **absent** from that file at `58d8d9754` (`db_ops.go` itself survives #1272 and is deleted at #1291). §5.8 now splits 37 into 27 ordinary #1291 deletions, 3 #1272 Vault-routing casualties mapped to their retained handler/Vault owners, and the 7 exceptions. The controller row was also wrong: `TestHandleLockReqErrors` covers the stopped-state error **only**, so the other two do not fold into it — `TestHandleLockReq_Idempotency` maps to `TestWalletVaultLockIdempotent`, and `TestHandleLockReq_LockError` is removed with its branch because `keyvault.Vault.Lock()` is void at the head (`keyvault.go:35`). **Partly superseded by §15.10:** the `TestHandleLockReq_Errors` rename was later reversed — the base name is restored under rule 1; the idempotency mapping and the deleted lock-error branch stand |
| gpt 4 — the normative block is not executable as written | **taken, and verified by running it.** `head=<exact candidate OID for the head under test>` is invalid shell, and `status` is a **read-only** parameter in zsh — this workspace's default shell, confirmed by `zsh -c 'status=$?'` failing. The head now arrives as a checked `$HEAD_OID` (so `set -u` closes the unbound path), the variable is renamed `grep_status`, and the section states the block runs under Bash. Extracted from the markdown, `bash -n` clean, executed against the real repository: exit 0, emitting the expected 51 for the rev-6 input audit |

### 15.9 Plan-review disposition (2026-08-01, eleventh round; historical)

This clean verdict is superseded by the four corrections in section 15.10.

Codex rechecked the complete current plan after the tenth-round corrections and
found no remaining executable or scope issue. The review independently
confirmed:

- the serialized inventories and the 11/51/51 rev-6 input audit;
- the 8/46/46 candidate expectation after the five same-name restorations;
- the complete 37-name `db_ops` split of 27 #1291 deletions, three #1272
  Vault-routing deletions, and seven individually mapped retained properties;
- the corrected controller, signer, syncer, and kvdb change-marking owners;
- the Bash inventory gate's bound candidate OID and propagated failures; and
- no new production behavior, itest, PostgreSQL goal, lifecycle hardening,
  atomic creation, migration, or unrelated bug-fix scope.

Two evidence-only wording errors were corrected in place: §15.7's obsolete
one-owner/no-change-owner account is now explicitly superseded, and §15.8 now
describes the actual 27+3+7 ownership split instead of calling all 30
non-exception names one group. No executable requirement changed.

### 15.10 Plan-review dispositions (2026-08-01, twelfth round)

GPT independently rechecked the executable plan after the eleventh-round clean
review. The owner authorized direct task-file corrections for any remaining
issues, so the four findings below were applied in place.

| finding | disposition |
| --- | --- |
| **blocker — the normative inventory invocation names a nonexistent script and can mask an output failure** | **taken.** Section 10.2 invoked `bash inventory-gate.sh`, but the task neither contains nor creates that file; prior verification worked only after a reviewer manually extracted the following block. The gate is now one directly runnable Bash here-document with a checked caller-bound `candidate_oid`. Cleanup uses an `EXIT` trap, and `cat` is status-checked so a failed output write cannot be hidden by successful cleanup |
| **high — two style-only test renames violate base-relative minimality** | **taken.** `TestHandleLockReq_Errors` has an unchanged stopped-state body, and the pinned-base body of `TestControllerStart_DBGetAllAccountsFail` already calls `Store.ListAccounts`; rev 6 changed only their names/comments. Restoring their base names is smaller than repairing pre-existing naming in this reduction. The idempotency and deleted-lock-error dispositions remain unchanged |
| **high — public-key child derivation lost its failure falsifier** | **taken.** The retained Store tests cover success, account absence, Store failure, and missing account xpub, but none reaches the branch/index failures still returned by `deriveStoredAccountChildPubKey`. `TestDerivePubKeyDeriveFails` is now restored and minimally adapted with a valid Store xpub and hardened public-child component. Together with the two name restorations, this changes candidate inventory expectations from 8/46/46 to **7/43/43** |
| **medium — the SQL selector source gate was named but undefined** | **taken.** `AccountID` cannot be prohibited globally because section 4.2 legitimately retains it. Section 4.1 now defines the gate as the exact `TestResolveScanTargetsDoesNotUseSQLAccountID` command plus exact-OID source review requiring imported-`AccountID` target lookup to be control-flow dominated by `s.addrStore != nil`; section 10.1 runs both at every applicable OID |

These corrections restore base coverage and make existing requirements
executable. They add no production behavior, integration scenario, backend
goal, or deferred feature.

### 15.11 Final plan-review disposition (2026-08-01, thirteenth round)

Codex re-read the active sections after the twelfth-round corrections and
Fable's independent final verification. The branch ancestry and 7/4/1 commit
shape still match the pinned OIDs; the normative inventory here-document runs
verbatim and reproduces the 11/51/51 rev-6 input audit; and its eight planned
same-name restorations give the internally consistent 7/43/43 candidate
expectation. The two controller restorations are base-preserving name reversals,
and the adapted `TestDerivePubKeyDeriveFails` is the missing falsifier for the
retained public-child branch/index error path.

One executable ambiguity was corrected: section 5.1's "at most" Legacy Vault
forwarding test allowed either deletion or retention despite section 5 calling
itself exact. It now requires one two-row table for the two retained explicit
key classes, without restoring the removed wallet-mode policy matrix. Two
record-only corrections align section 15.8 with the final three-test
change-marking ledger and mark section 15.9's obsolete 8/46/46 verdict as
historical.

No production behavior, integration scenario, PostgreSQL goal, lifecycle
hardening, atomic creation, migration, or unrelated bug fix was added. Task 316
is ready to execute. The next review target is the rebuilt candidate history,
not another plan revision.

### 15.12 Source files

- `gpt-review-task.md`: the rev-6 scope and test audit, **plus** its Task-316
  plan-review rounds through the twelfth, whose corrections §15.10 applied;
- `codex-review-task.md`: the rev-6 implementation review, **plus** its
  Task-316 plan-review rounds through the thirteenth;
- `fable-review-task.md`: Task-316 plan review, rev-6 spot checks, and its final
  verification of the twelfth-round corrections;
- `opus-report.md` section 11: rev-6 implementation evidence and known
  overclaims;
- Task 315 sections 1, 5.4-5.8, and DoD 4-13.
