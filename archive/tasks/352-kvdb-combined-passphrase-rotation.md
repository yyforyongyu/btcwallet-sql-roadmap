# Task 352 — contain uncertain kvdb passphrase rotation

Component: **Controller and LegacyWalletVault**.

Type: **Bug fix**.

Dependencies: **[Task 331](331-manager-wallet-lifecycle.md) and
[Task 467](467-indeterminate-commit-identity.md)**.

## Archive decision

Deferred on 2026-09-21 as kvdb-only work, not completed or obsolete.
Reactivate this number only for an explicitly approved future-cycle kvdb
passphrase-containment effort. SQL Vault work proceeds without this gate;
its dependent kvdb refactoring stays deferred.

## Rationale

The maintained `Controller.ChangePassphrase` can ask the modern kvdb Vault to
rotate public and private credentials together. `LegacyWalletVault` calls
`waddrmgr.Manager.ChangePassphrase` twice inside one outer `walletdb.Update`,
but each successful call replaces its in-memory key before that outer
transaction commits. A reachable wrong-private-credential or bbolt commit
failure can therefore return an ordinary error while the live legacy manager
no longer matches the durable credential state.

The Wallet-owned adapter cannot delay or restore private `waddrmgr` state. It
must contain that uncertainty instead of claiming atomic disk-and-memory
rotation or leaving later credential operations free to use the stale manager.

## Details

- Limit the change to maintained `Controller.ChangePassphrase` and its
  Wallet-owned `LegacyWalletVault` implementation. Preserve the legacy dual-
  credential format and modern kvdb's one-Wallet limit.
- Serialize `Unlock`, `Lock`, `Encrypt`, `Decrypt`, `ChangePassphrase`,
  `IsLocked`, and later typed secret operations through one
  `LegacyWalletVault`-owned operation guard. Do not add a generic Controller
  operation registry or Store-wide write lock.
- Enforce Store-before-Vault ordering. A method that needs walletdb opens its
  existing transaction before taking the guard; `ChangePassphrase` takes the
  guard inside `walletdb.Update` and retains it through commit classification
  and terminal latching. Never open a walletdb transaction while holding the
  guard. Methods without Store access take only the guard.
- Keep the existing successful path: call the unchanged
  `waddrmgr.Manager.ChangePassphrase` operations inside one
  `walletdb.Update`. If every requested half and the outer commit succeed,
  preserve the initial locked or unlocked state and return success.
- Treat an error before any legacy passphrase call succeeds as a determinate
  request failure. It leaves the adapter usable and preserves the existing
  translated error identity.
- After any legacy passphrase call succeeds, classify a later credential-half
  error or outer commit error as the public, wallet-owned
  `ErrWalletReopenRequired` identity, detectable with `errors.Is`. Do not
  report whether the old or new credential is durable or retry the mutation.
- If the outer commit outcome cannot be proven, the result must also match
  Task 467's `ErrIndeterminateCommit` through `errors.Is`. A determinate
  second-half failure and rollback match only `ErrWalletReopenRequired`.
- Latch that terminal result in `LegacyWalletVault`. Later credential-use and
  mutation operations fail with it before consulting the uncertain manager.
  Keep serialized `Lock` and observational `IsLocked` available: Lock may
  clear in-memory private material because it does not read or choose durable
  credential state.
- Require the caller to run final `Manager.Stop`, which locks the Vault and
  joins Wallet work, before constructing a fresh Manager and selecting the
  reopened Wallet from `Manager.Start`. Do not claim the Wallet can initiate
  its own stop or remains safe for credential-dependent work.
- Leave response latching and cancellation after Controller acceptance to
  [Task 338](../../tasks/338-passphrase-result.md).
- Do not modify `waddrmgr`, `walletdb`, Loader, deprecated Wallet methods, or
  their tests. Do not add a rotation plan, compatibility adapter, generic
  lifecycle state, cross-Manager fencing, or multi-process guarantee.
- Modern kvdb remains maintained for the current and next release. Deferred
  retirement does not eliminate this correction.

## Suggested fix

Add one operation guard and terminal result latch to `LegacyWalletVault`.
For Store-bearing methods, enter the existing walletdb transaction before the
guard. Hold the guard from the `ChangePassphrase` callback through its commit
classification and latch. Track whether an unchanged legacy passphrase call
has completed successfully; on any later error, retain and return
`ErrWalletReopenRequired`, also preserving `ErrIndeterminateCommit` for an
unprovable outer commit. Reject subsequent credential work from the latch
while allowing Manager Stop's Lock to wipe the old manager before durable
reopen.

## Verification

- Drive `Controller.ChangePassphrase` with valid public and invalid private
  credentials. Prove the public half can succeed in memory before the outer
  transaction rolls back, the call returns `ErrWalletReopenRequired` through
  `errors.Is`, and later credential operations do not consult the legacy
  manager.
- Inject outer bbolt commit outcomes after one or both successful halves. A
  proven rollback matches only `ErrWalletReopenRequired`; an unprovable result
  matches both that identity and `ErrIndeterminateCommit` without claiming old
  or new. After Manager Stop, a fresh Manager reads whichever tuple committed.
- Fail validation and the first legacy call before any successful mutation and
  prove the ordinary error leaves the adapter usable. Cover public-only,
  private-only, and combined successful rotation without changing lock state.
- Race Lock, Unlock, Encrypt, Decrypt, and rotation. The operation guard admits
  no credential operation between the two legacy calls, and the terminal latch
  prevents one after an uncertain result.
- Use barriers to race modern-kvdb `AccountManager.NewAccount`, while its Store
  write calls Vault encryption, against `Controller.ChangePassphrase`. Prove
  Store-before-Vault ordering lets both complete without admitting credential
  work between the two legacy calls, commit classification, and terminal
  latching.
- Prove Manager Stop can still call Lock, clear the uncertain manager's private
  material, and complete before a fresh Manager starts.

## Definition of done

- Maintained modern-kvdb passphrase rotation either succeeds coherently,
  returns a determinate pre-mutation error, or returns
  `ErrWalletReopenRequired` and rejects later credential work on that adapter;
  an unprovable commit also matches `ErrIndeterminateCommit`.
- Manager Stop can wipe the uncertain manager, and only a fresh Manager reopen
  resumes credential-dependent Wallet use.
- Store-bearing Vault operations acquire Store before the operation guard, and
  concurrent account creation and rotation complete without deadlock.
- The PR changes no `waddrmgr`, `walletdb`, Loader, deprecated Wallet method or
  test, generic registry, broad lifecycle, or cross-process behavior.
