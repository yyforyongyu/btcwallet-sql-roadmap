# Task 308 — finish the three PRs so itest work can proceed

Status: **NON-EXECUTABLE — historical record, superseded by [[Task 313]].**

> **Do not execute anything in this file.** Every imperative section below —
> including the replay cascade and the `--force-with-lease` push commands — is
> archived. It describes the round-1 execution that produced
> `0d98a7d45`/`8b4f29f74`/`1243c274c`, against a premise that no longer holds.
>
> Its **"Revision 2"** is superseded in particular: the constant
> `InsecurePubPassphrase` kvdb design is *not* single encryption, its lifecycle
> addition keeps wallet-owned Stores, and its atomic-create addition excludes
> canonical kvdb. See [[Task 300-INDEX]] for the replacement chain.
>
> Useful only as evidence: the cascade mechanics, the token-audit method, and
> the round-1 verification record.

## Goal, And The Standard It Sets

itest work is blocked on #1272, #1291 and #1292. The goal is to get those three
into a mergeable state — nothing more. **Issues discovered along the way are
recorded and fixed in future PRs**, not fixed here. That standard is what makes
this round small.

Two consequences worth stating up front, because they remove work I previously
treated as mandatory:

- **Atomic SQL creation is no longer on the critical path.** Dropping the create
  manifest leaves creation non-atomic. That is a *known issue for a future PR*
  ([[Task 1000]]'s `BWR-013` territory), not a blocker — tests do not
  crash-inject. #1291 becomes a truncation and little else.
- **`6d414012b` is not cherry-picked.** Its sync-tip hardening is already
  described in the [[Task 1000]] reference (`BWR-005`) — described, not
  scheduled, and 1000 is not an owner. The cherry-pick list drops to one commit
  and "no commit surgery" becomes literally true.

## Scope

`wallet/` only, as a library for consumers like lnd. The public API is the
component interfaces — `Controller`, `Manager`, `AccountManager`,
`AddressManager`, `TxReader`/`TxWriter`, `TxCreator`, `TxPublisher`,
`Signer`/`UnsafeSigner`, `UtxoManager`, `PsbtManager` — each with a compile-time
`var _ X = (*Wallet)(nil)` assertion.

Out of scope and not a compatibility constraint: `wallet/deprecated.go`,
`wallet/interface.go`'s aggregate `Interface`, `rpc/`, `legacyrpc`, and every
other in-repo consumer.

## Execution Cascade — Do This In This Order

**Why this section exists.** [[Task 305]] rewrites #1272 **from commit #2**, so
every descendant SHA changes. That makes "cut #1291 at `1047a9b91`" not
executable on its own: verified, `1047a9b91`'s ancestry runs through the *old*
#1272 (`7bf664f83` -> `ee410d0e9` -> `d8b56f7db`). 305 used to hand descendant
repair to Task 306, which is now retired — so this task owns the replay. Without
it the plan cannot reach its own mergeable-stack goal.

1. **Back up all three heads** before touching anything. An earlier draft named
   `refs/backup/1291-full/`, which **git rejects** — a ref name cannot end in
   `/` (`git check-ref-format` returns INVALID). Use three valid, SHA-pinned
   refs:

   ```
   git update-ref refs/backup/t308/impl-secret-store      7bf664f83
   git update-ref refs/backup/t308/impl-runtime-store-6   17638808c
   git update-ref refs/backup/t308/wallet-default-sqlite  d98e82bab
   ```

   Verify all three resolve before rewriting anything.
2. **Rewrite and push #1272** per [[Task 305]]'s rescoped set.
3. **Replay the nine retained #1291 commits onto the new #1272, in order** —
   by subject, not by SHA, since the SHAs change:

   ```
   9fd21f32a kvdb: add runtime store lifecycle adapters
   254c2de6c waddrmgr: export birthday safety margin
   9d9f68576 wallet: add runtime store config
   c32b4f69e wallet: add runtime store open helpers
   bd93fc4c2 wallet: coordinate manager publication
   676c5d350 wallet: create through runtime stores
   666374c9a wallet: route sync runtime through Store
   8f03737f0 wallet: complete runtime store migration
   1047a9b91 itest: wire manager DB config to runtime store
   ```

4. **Apply `a567be720`** (`wallet: re-export TxStatus for external callers`).
   While replaying `wallet: complete runtime store migration`, also correct the
   `isKVDB` comment at `wallet/manager.go:1463-1470`: it says rotation "stays on
   `Wallet.ChangePassphrases`", which F0 makes false. The cut **keeps** that
   comment, and an earlier plan routed the fix to retired Task 306 — it is
   documentation in retained code made wrong by the required #1272 rewrite, so
   it belongs in this replay.
5. **Replay #1292's two commits onto the new #1291** —
   `a4ad6a1a6 wallet: default new wallets to the sqlite backend` and
   `d98e82bab bwtest: default itests to sqlite, reject postgres` — including the
   register-before-`Start` correction.
6. **Verify and push each head in dependency order**, then record the three
   final SHAs here. Each push is a force-rewrite, so use
   `--force-with-lease=<branch>:<old-head>` pinned to the backed-up SHA above —
   never a bare `--force`:

   **The PR heads are on `upstream`, so that push is the one that matters.**
   Verified via `gh pr view`: all three PRs have
   `head=btcsuite/btcwallet:<branch>`, and in this checkout `upstream` is
   `btcsuite/btcwallet` while `origin` is the `yyforyongyu/btcwallet` fork. Only
   the `upstream` push updates a PR.

   So push **`upstream` first, per branch**, and only then mirror to `origin` to
   keep the fork in sync. That ordering matters: doing the fork first risks a
   succeeded-fork/failed-PR state that reads as done but has not moved any PR.
   Use full SHAs and explicit refspecs — no placeholders:

   ```
   # PR head first — this is the push that updates the PR.
   git push --force-with-lease=impl-secret-store:7bf664f8361362e6b4fbc1426c1eaa7685027263 \
     upstream refs/heads/impl-secret-store:refs/heads/impl-secret-store
   # then mirror the fork
   git push --force-with-lease=impl-secret-store:7bf664f8361362e6b4fbc1426c1eaa7685027263 \
     origin refs/heads/impl-secret-store:refs/heads/impl-secret-store
   ```

   Repeat for `impl-runtime-store-6` (lease `17638808cef6b3fc8114fac0fe43c493eb9f6c49`)
   and `wallet-default-sqlite` (lease `d98e82bab173715c5f07b1f391cb6a10283fe3c2`).

   The `origin` mirror is a convention of this repo, not a requirement of the
   PRs; if a fork push fails it is safe to retry separately, because the PR is
   already updated.

   Push one branch at a time in dependency order, not the whole stack at once:
   a rapid whole-stack force-push has previously dropped CI on some PRs.
7. **Hand #1300's owner the final #1292 SHA** plus the old/new base range. Do
   **not** rebase #1300 in this task — it is the immediate next dependent, not
   deprioritised.

## The Three PRs

### #1292 `wallet-default-sqlite`

1. **Register before `Start`, in both sites.**
   `bwtest/harness_wallet.go:112-127` and `itest/manager_test.go:44-55` both do
   `Create` → `Start` (under
   `require.NoError`) → `RegisterWallet`. A failed `Start` exits the test before
   the harness owns the Manager-opened store, leaking it past
   `stopActiveWallets` and poisoning later subtests. Safe because `Stop` on a
   never-started wallet logs and returns nil. This is thread `UfRny`.
2. **Make the SQLite path available** — an explicit `DBConfig` with
   `DBBackendSQLite` and a zero-valued kvdb config, expressible through the
   public API. Context: no workflow passes `db=sqlite` today (`main.yml:241`,
   `:287`, `:356` are all `db=kvdb`) even though the harness defaults to it
   (`967eeb3b7`). **Adding a CI job or choosing cases is not part of this
   round** — that is #1300's and the future evidence milestone's work.

### #1291 `impl-runtime-store-6` — truncate

**Cut at `1047a9b91`**, then cherry-pick **`a567be720`** whole.

Verified clean at the cut — every token zero under `wallet/`, and the three
files absent:

| | at `1047a9b91` | at full #1291 |
| --- | --- | --- |
| `create_manifest` / `CreateManifest` | 0 / 0 | 8 / 26 |
| `create_complete` / `CreateComplete` | 0 / 0 | 8 / 30 |
| `SetWalletCreateComplete` | 0 | 12 |
| `WriteGate` / `writeGate` | 0 / 0 | 4 / 9 |
| `chainResetCoordinator` / `ResetChain` | 0 / 0 | 5 / 13 |
| `runtimeStoreRegistry` | 0 | 3 |
| the three machinery files | absent | present |
| `migrations/000011_*` | 0 files | 4 files |

Size: **51 files, +8,685/−3,492** from the #1272 head, versus +19,571/−3,997 —
a 56% reduction, and the eight dropped commits are 42% of the insertions.

**Why `a567be720` (+75/−2) is the one exception.** At the cut,
`TxDetail.Status` is `db.TxStatus` (`tx_reader.go:147`) — a type from
`wallet/internal/db` that external consumers **cannot import**, so `TxReader`
would ship a field nobody can name, compare or switch on. That is a regression
in the deliverable rather than a pre-existing issue, so it is cheaper to keep
than to schedule. It also restores the convention the rest of the library
follows (`utxo_manager.go:36-49` translating internal sentinels into public
`wallet.Err*` values).

**Why not `483a6dc9a` as the cut instead:** it inherits the create-completion
surface (8–10 files plus migration `000011`) because `b63c480d8` lands below it,
and `483a6dc9a` needs that commit's UTXO query widening (`account_id`,
`purpose`, `coin_type`, `account_name`) for its `(KeyScope, AccountID)` scoping.
Using it would require splitting a generated commit. Withdrawn.

**Rejected outright by the owner — never merged:** `e34098cc1` (create
completion), `938dc2269` (create manifest), `07ac7039b` (shared runtime
registry), `17638808c` (reset coordinator + `WriteGate`).

**Permanently dropped, not deferred:** `b63c480d8` — it carries the generated
create-state surface (`SetWalletCreateComplete`, migration `000011`) for the
design the owner rejected. If its UTXO query widening is wanted later it gets
reimplemented under its own owner, not resurrected from this commit.

**Deferred but valid — future PRs:** `552c2849d` (`Manager.Load`
serialization), `483a6dc9a` (targeted `Rescan` scoping), `6d414012b` (sync-tip
guards).

Bookkeeping: the three backup refs are created in cascade step 1 — the full
pre-cut #1291 is preserved at `refs/backup/t308/impl-runtime-store-6`. Update
the pre-rewrite head tables in [[Task 305]], and notify @GustavoStingelin that
#1300's base moved.

### #1272 `impl-secret-store`

Per [[Task 305]], rescoped this round: the **F0 revert** (which removes code),
**F1** `Controller.ChangePassphrase` accepted-request completion, plus **F5**
(the red `-race` test blocks the merge), **F6**, **T7** and **F9**. F2, F3, F4
and F7 are legacy-path work and are **not** gates — see 305 for the dispositions
and the thread replies.

## Known Limitations To Record In The #1291 PR Body

These are accepted for this round and each needs a tracking item, so they are
visible rather than rediscovered:

1. **Creation is not atomic** — a crash mid-create can leave a partial wallet;
   the manifest that used to detect it is gone by design.
2. **Per-wallet SQL pools** — no shared registry, so two wallets on one SQLite
   file collide. One wallet per file for now.
3. **No multi-wallet reset coordination** — the rollback-versus-sync race is
   open, because per-wallet syncers survive the cut. Single-wallet reorg remains
   self-serialized inside that wallet's syncer.
4. **No `Manager.Load` serialization** and **no targeted-`Rescan` scoping**.
5. **Public types still leak internals** — `AccountManager` returns
   `db.AccountInfo` on every method (`account_manager.go:118`, `:121`, `:126`,
   `:132`), with nested `db.KeyScope`/`db.ScopeAddrSchema`. This is `BWR-001`.

## The Deferred Limitations Are Informational

**This round does not assign future owners, and does not need to.** An earlier
version of this section demanded "exactly one future owner" per limitation and
then listed several as unowned — a rule it could not satisfy. The standard that
actually applies: **the #1291 PR-body entry plus its code `TODO` is the tracking
record** until a future PR claims the issue. Creating tasks or minting IDs now
would be inventing owners for work nobody has scheduled.

[[Task 1000]] is **reference only** — not worked on, not edited, and **not an
owner**. Consult it before proposing a follow-up, purely to avoid duplicating
something already described:

| Limitation | Described in 1000 as (non-normative) |
| --- | --- |
| Creation not atomic; commit ambiguity | `BWR-013` |
| Per-wallet SQL pools / runtime ownership | `BWR-000` |
| Multi-wallet reset / rollback-vs-sync | `BWR-005` |
| `Manager.Load` serialization | `BWR-001` |
| Targeted `Rescan` scoping | `BWR-005` |
| `AccountInfo` leaks `db.*` types | `BWR-001` |

`TxStatus` is **not** in that list: `a567be720` closes it in #1291, so it is no
longer a deferred leak even though 1000 groups it with `AccountInfo`.

**Two caveats about 1000 before anyone picks it up**, recorded here rather than
fixed there: its *Revalidated Scope* says to **preserve** the create manifest
and completion state, and `BWR-013` prescribes manifest v2,
`create_complete=false`,
resumable prefixes and `create_state` (`:1487`, `:1508`, `:2914`, `:2921`,
`:2934`, `:3540`), while `:2707` and `:428` still assume `WriteGate` exists. All
of that predates the decision to remove that machinery and needs re-validation.

## Code TODOs To Add

A `TODO(yy)` goes where a reader of that code would need the warning — at the
narrowest true location, naming the **limitation**, never a `BWR-` id this round
does not own. Not one per row as a checklist: a TODO buried where nobody reads
it is churn, and reviewers pushed back on making it a blanket requirement.

**No TODO for F2.** Its two defects are in `Wallet.ChangePassphrases`, a
deprecated method outside the library API. Adding a marker there would open a
tracking surface for exactly the code this round declared out of scope; the
thread answers record it instead.

## Relationship To Task 1000

[[Task 1000]] is the goal-level plan for the same objective — *"the
btcwallet-owned work required for a fresh lnd node to use the role-based SQLite
or PostgreSQL wallet"* — and it plans **8 PRs of its own** (`PR 1`–`PR 8`) with
a 15-item ledger (`BWR-000`–`BWR-014`).

**It is reference only. This round does not work on it and does not edit it.**
Its value here is showing that several issues surfaced this round are already
understood, so follow-ups need not be re-derived. In one case its approach beats
what the retired Task 306 proposed:

| Item | 1000's entry |
| --- | --- |
| `AccountInfo`/`TxStatus` type leak | `BWR-001` High, + compile test |
| `MaxConnections=1` bootstrap deadlock | `BWR-000` conn-bound executor |
| Ambiguous-commit reconciliation | `BWR-013` (Critical) |
| Synced tip preceding transaction state | `BWR-005` (High) |

## Definition Of Done

1. Registration happens before `Start` in `bwtest/harness_wallet.go` and
   `itest/manager_test.go`.
2. The public SQLite path is **constructible**: a `DBConfig` with
   `DBBackendSQLite` and a zero-valued kvdb config wires a wallet through the
   public API, and the changed packages build. **No CI job, chain backend, or
   `make itest` invocation is a gate this round** — #1300 owns cases and the
   execution matrix.
3. #1291 is cut at `1047a9b91` plus `a567be720`, and the narrowed branch
   contains **zero** occurrences of `create_complete`, `CreateComplete`,
   `SetWalletCreateComplete`, `create_manifest`, `CreateManifest`,
   `runtimeStoreRegistry`, `WriteGate`, `writeGate`, `chainResetCoordinator` or
   `ResetChain`, and no `migrations/000011_*`. Match those **exact** tokens: an
   earlier verification used only the Go camelCase identifier `createComplete`
   and reported a clean tree that in fact carried the SQL and generated
   surface.
4. **All thirteen unresolved threads have a disposition** — each either fixed,
   or answered with a reason and an explicit deferral. **No owner is required**;
   "valid finding, deferred from these three PRs, not yet assigned" is a
   complete answer. Seven on #1272, five on #1291, one on
   #1292; an earlier draft of this DoD covered only #1272's:
   - **#1272 (7)** — `Ue1m1`/`Ue2Iz`/`Ue9uZ` closed by F0; `Ue9wZ`/`UfRnj`
     answered as deprecated-surface (dropped F2); `UfRnp` by F6; `UfRn0` by T7.
   - **#1291 (5)** — `UfRns` (dead `scanTarget.AccountName`) and `UfSB5`
     (`InitialAccounts` doc) are one-line fixes in code the cut **keeps**, so
     fix them. `UgVTm` (stop kvdb create), `UgPWI` (`IsMissingStore`) and
   `UgRON`
     (`Store.DB`) asked for deletions that were Task 306's. Answer them
     "valid finding, accepted and deferred from these three PRs, not yet
     assigned" — do **not** invent an owner or cite 1000's ledger as accepting
     them.
   - **#1292 (1)** — `UfRny`, fixed by DoD 1.
5. All three PRs build and their existing tests pass; each commit passes
   `go vet ./...`.
6. The known-limitations list is in the #1291 PR body, and each limitation a
   reader could trip over has a code `TODO` naming the limitation. No `BWR-`
   assignment is required or implied.
7. The three backup refs above exist at their pinned SHAs before any rewrite,
   each final head descends from its predecessor, and every push used
   `--force-with-lease` against the recorded old head.

---

# Revision 2 — Single-Encryption Doctrine (2026-07-30)

Status: **ARCHIVED — superseded by [[Task 313]]. Not the plan of record.**
Its constant `InsecurePubPassphrase` kvdb design is not single encryption, its
lifecycle addition keeps wallet-owned Stores, and its atomic-create addition
excludes canonical kvdb. Do not execute anything below.

Historical note — the round-1 execution above
shipped; the owner then set a further directive that reshapes the passphrase
surface and sizes the PRs down. Round-1 sections stay as the record of what
happened; where they conflict with this revision, this revision governs.

## The Doctrine

**All backends assume single encryption — kvdb and SQL alike.** The user's
passphrase is the private passphrase, full stop.

- **New kvdb wallets are created single-encryption**: `Manager.Create`'s kvdb
  path hardcodes `pubPass = wallet.InsecurePubPassphrase` ("public" — already
  the standalone daemon's default). The public half is a non-secret constant,
  which is the same at-rest stance SQL takes by storing public rows in
  plaintext. It must be the constant, not `pub = priv`: the wallet reads
  addresses while locked, so the public half must open without the user.
- **Existing dual-passphrase kvdb wallets are deferred** — opening and
  migrating them is a future PR, not this round (see "Deferred" below).

## What This Deletes From The Round-1 Branches

1. **The atomic dual-rotation machinery** (#10 `waddrmgr+wallet: rotate
   passphrases atomically`, +741): `waddrmgr.ChangePassphrases`, the staged
   commit apparatus, the locker wiring, the `atomicPassphraseRotator`
   extension interface and its TODO. `wallet/deprecated.go` reverts to its
   pre-stack form. **This moots F4 (the partial-zero gap lives in deleted
   code), F5's rotation race tests, T7 entirely, and thread `UfRn0`.**
2. **The dual-passphrase API**: `ChangePassphraseRequest` becomes
   `{Old, New}` (delete `ChangePublic`/`PublicOld`/`PublicNew`);
   `keyvault.ChangePassphraseParams` stays narrow;
   `ErrPublicPassphraseUnsupported` is unnecessary (nothing to reject) — F9
   reduces to the `wallet.ErrInvalidPassphrase` alias.
3. **`LegacyManagerVault` dual delegation** (#13, +352): becomes private-only
   delegation via the pre-existing single-half `waddrmgr.ChangePassphrase`
   (~15 lines plus test).
4. **`PubPassphrase` off the public API**: removed from `CreateWalletParams`
   and `Config`; `PrivatePassphrase` renamed `Passphrase` (the API is
   unmerged; the rename is free and states the doctrine).
5. **`ScriptIsSecret` off the store contract**: key-type selection is buried
   at the kvdb boundary (the adapter already holds waddrmgr's per-address
   flag internally; it selects the crypto key instead of exporting the
   flag). `scriptCryptoKeyType`, the signer branching, and
   `encryptTaprootScript`'s watch-only key split go with it. One
   implementation choice, decided at the call site: adapter-internal
   selection (preferred, no data movement) versus re-encrypt-at-unlock.
6. **Public struct trim** (minimal-fields directive): `AddressInfo` drops
   `HasScript`/`Imported`/`Internal`/`Compressed` (zero readers; all
   derivable — verified); `AddressDerivation` collapses
   `Account uint32 + HasAccountNumber bool` into `AccountNumber *uint32`,
   matching the store layer and the sibling `AccountID *uint32`. The two
   path-emission gates become `Derivation == nil` /
   `Derivation.AccountNumber == nil`.
7. **F3 collapses to one shape**: private-only rotation through
   `Controller.ChangePassphrase` over the kvdb fixture and SQL. The
   unequal-shape ("public"/"hello") coverage moves to the future migration
   PR, where it belongs.

Estimated effect: #1272 goes from +9,659 to roughly +7,000-7,200 insertions,
with five public fields, one contract field, one exported method family, and
one sentinel gone. #1291 sheds only the `PubPassphrase` threading; #1292 is
unchanged.

## Deferred — The Thin kvdb Migration (Future PR, Recorded Here)

Convert existing dual-encryption kvdb wallets to single **after unlock**,
when the private passphrase is in hand:

- try the private passphrase as the public one (lnd-created wallets have
  pub == priv); on success, rotate pub → `InsecurePubPassphrase` with the
  single-half `ChangePassphrase` in one walletdb transaction, gated by a
  waddrmgr manager-version bump, idempotent;
- two tails need a supplied old public passphrase instead: custom
  `--walletpass` wallets (pub ≠ priv ≠ constant) and watch-only kvdb wallets
  (no unlock ever runs);
- until it lands, a pub ≠ "public" kvdb wallet cannot `Manager.Load`. Cost
  today: zero — the standalone default is the constant, and lnd is not on
  the new API. Record this in the #1272 PR body as a known limitation.

The final kvdb→SQL migration remains a separate, later milestone.

## Thread Re-Answers

- `Ue1m1`/`Ue2Iz`/`Ue9uZ`: closed at the root — single-passphrase model;
  the refusal/redirect dead end no longer exists in any form.
- `UfRn0`: closed by deletion — nothing is added to `AddrStore` and no
  workaround interface exists.
- `Ue9wZ`/`UfRnj`: unchanged (deprecated surface, answered as before).

## Execution Cascade — Revision 2

1. Back up the round-1 heads: `refs/backup/t308r2/{impl-secret-store @
   0d98a7d45, impl-runtime-store-6 @ 8b4f29f74, wallet-default-sqlite @
   1243c274c}`; verify they resolve.
2. Re-rewrite #1272: reshape #10-#13/#17 per the deletion list; add the
   single-encryption creation change (constant pub in the kvdb create path)
   and the API renames/trims; keep the account-secret and signer-through-
   store commits as they are.
3. Replay the nine #1291 commits + the `TxStatus` re-export onto the new
   #1272 (same subjects as round 1), shedding `PubPassphrase` threading
   where it appears.
4. Replay #1292's two commits.
5. Verify per commit (`go vet ./...`), branch tests, lint at each head; push
   one branch at a time to both remotes with
   `--force-with-lease=<branch>:<round-1 head>`.
6. Re-answer the three re-closed threads; add the deferred-migration
   limitation to the #1272 PR body; hand #1300 the final #1292 SHA.

## Definition Of Done — Revision 2

1. No dual-passphrase surface exists on any branch: zero occurrences of
   `ChangePublic`, `PublicOld`, `PublicNew`, `ChangePassphrases(`,
   `atomicPassphraseRotator`, `ErrPublicPassphraseUnsupported`,
   `ScriptIsSecret`, or `PubPassphrase` in non-deprecated `wallet/` code
   (deprecated.go keeps its historical pre-stack surface).
2. `Manager.Create` kvdb wallets are created with the constant public
   passphrase and open/load/unlock/rotate through the single-passphrase API;
   the kvdb fixture test proves it end to end.
3. `AddressInfo`/`AddressDerivation` carry the trimmed field set; the two
   path-emission gates use presence, not flags.
4. All three PRs build, tests pass, CI green; threads re-answered; the
   deferred-migration limitation is in the #1272 PR body with its TODO.
5. `refs/backup/t308r2/*` exist at the round-1 heads before any rewrite;
   every push used `--force-with-lease` against those heads.

## Revision 2 Additions (owner Q&A, 2026-07-30)

8. **Atomic SQL creation in one store transaction — decided.** The split
   creation sequence is the root cause of `manager.go`'s retry/recovery
   surface (`prepareWalletCreation`, `verifyRootKeyMatchesLegacy`,
   `recoverPersistedMasterRoot`/`recoverRuntimeMasterRoot`/
   `recoverLegacyMasterRoot`, `decryptLegacyMasterRoot`) — #27 alone adds
   +1,170 lines there. Fix at the root: prepare everything outside the
   transaction (validate, derive, encrypt; zero temporary keys after), then
   one `Store.CreateWallet(aggregate)` executing every insert in a single
   `sql.Tx` via the existing `dbruntime.Write` plumbing — wallet row,
   secrets, key scopes, default/imported accounts, account secrets, commit.
   A failed transaction leaves no wallet; a crash after commit leaves a
   complete wallet. Deletes the retry-authentication family and the
   partial-wallet handling; removes "creation is not atomic" from the
   known-limitations list and its TODO. kvdb creation is already atomic
   inside its own walletdb transaction and gains the constant public
   passphrase per item 1 of this revision.
9. **Lifecycle simplification — proposed, owner to confirm before
   executing.** #27's `controller.go` +407 is teardown-ownership
   choreography (four-way stop claims, deferred-stop intents,
   `finishStopAsync`, `waitForStopped` polling, `stopResult`,
   abort-start teardowns) required by wallet-owned stores plus a lock-free
   style. Replace with a coarse per-wallet lifecycle mutex: Start and Stop
   each hold it for their duration; a Stop during Start waits, then stops;
   keep the per-name Load guard, vault-lock-on-stop, close-once, and
   no-restart-after-stop. Behavioral change to state in the PR: Stop blocks
   until an in-flight Start completes instead of aborting it midway. The
   deeper alternative (Manager-owned pool; Stop never closes stores) remains
   deferred and compatible. Estimated: −300 lines in `controller.go`, a
   smaller `state.go`, and materially less reviewer load on #27.

DoD (Revision 2) additions:

6. SQL wallet creation is one store transaction; the retry-authentication
   family is deleted; an injected failure at each insert step leaves no
   wallet row; the known-limitations list no longer contains the
   non-atomic-creation item.
7. If item 9 is confirmed: the stop-claim machinery is replaced by the
   lifecycle mutex, with the Stop-during-Start semantics documented on
   `Controller.Stop`.
