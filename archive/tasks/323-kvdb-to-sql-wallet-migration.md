# Task 323 — migrate existing kvdb wallets to SQL

**Status:** deferred, not scheduled. The terminal plan for kvdb.
Split out of [[archive/315-rebuild-minimal-dual-backend-itest-stack.md]] (archived) §4.

## Why deferred

Migration has been out of scope by owner ruling throughout this work, and
nothing needs it to land: the modern `Manager` serves new wallets on either
backend, and the deprecated `Loader` keeps opening existing kvdb wallets
unchanged. No user is stranded in the meantime.

## Why kvdb is **not** getting a new on-disk format

Recorded here because it is the decision this task supersedes.

Earlier plans (archived tasks 311 and 313) had new kvdb wallets adopt a
single-encryption waddrmgr format so both backends would share one key model.
315 keeps kvdb on the existing dual format instead, and that is now the end
state, not a deferral. The reasons:

- the architectural motive evaporated. `LegacyWalletVault` satisfies the typed
  `keyvault.Vault` contract over the **dual** format at the cost of one internal
  bit, `ScriptIsSecret` — see 315 §5.2. (An earlier revision claimed the cost
  was zero because `HasScriptKey()` made routing format-blind; that was wrong,
  since waddrmgr chooses the script key per address. One plumbed bit is still
  far less than a new format.) Uniform key handling never required a uniform
  disk format;
- a new format means writing two migrations for a backend being retired:
  dual → single, then single → SQL. Keeping one kvdb format means writing one,
  which is this task;
- the format needed a persisted discriminator, and `mgrver` cannot serve as one
  (`migration.Upgrade` runs credential-free before `waddrmgr.Open`, and the
  engine stamps the version itself), so it needed a **new** marker — permanent
  complexity in the layer being deleted.

The public API follows: `PubPassphrase` stays while kvdb is supported, and the
single-passphrase collapse happens when kvdb is dropped, not before.

## What to do

One converter, kvdb dual → SQL:

- read the waddrmgr and wtxmgr namespaces of an existing wallet;
- re-encrypt secrets under the SQL single-key model derived from the user's
  existing private passphrase — the public passphrase is discarded, since SQL
  has no equivalent. **Each script must be decrypted under the key its own
  persisted `isSecretScript` bit selects** (`CKTScript` when true, `CKTPublic`
  when false) before re-encryption; a whole-manager choice corrupts one of the
  two kinds;
- write accounts, addresses, transactions, UTXOs, and sync state through the
  Store contract;
- verify the result against the source before the source is retired, and never
  destroy the bbolt file — leave it in place for rollback.

Decide separately whether this runs on demand or automatically at open, and
whether `wtxmgr` history is migrated or the wallet simply rescans.

## Definition of done

1. A real wallet with accounts, imported addresses, scripts, and transaction
   history round-trips to SQL with balances, addresses, and sync tip identical.
2. The source bbolt file is untouched by the migration.
3. Watch-only wallets migrate too.
4. The deprecated `Loader` path is unaffected.

## Evidence

`archive/311-single-encryption-for-all-backends.md` and
`archive/313-single-encryption-revision-of-the-three-prs.md` describe the
format-unification design this task replaces; `archive/300-INDEX.md` records the
`mgrver` trap as verified.

## Carried forward from the delivered stack (codex, 2026-08-02)

**kvdb combined rotation can diverge in live memory.** Rotating both passphrase
halves applies each half separately; a failure between them can leave the
in-memory manager disagreeing with disk. Inherited from the base dual-passphrase
implementation, not introduced by the SQL work. The fix is a combined waddrmgr
rotation that prepares both halves before committing either.

**kvdb creation retains two public-passphrase inputs.** `Config.PubPassphrase`
(Load) and `CreateWalletParams.PubPassphrase` (Create) both survive because only
the retiring backend has a public half. They retire with kvdb itself.
