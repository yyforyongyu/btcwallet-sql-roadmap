# Task 352 — make kvdb combined rotation atomic

Component: **LegacyWalletVault, waddrmgr, and modern-kvdb AccountManager**.

Dependencies: **None**.

## Rationale

The modern kvdb Vault rotates the legacy public and private passphrases through
two manager calls inside one database transaction. If the second call fails,
the database rolls back but the manager's first in-memory key change can remain.

## Details

- Keep kvdb's existing dual-encryption format and one-wallet limit.
- Limit the fix to `LegacyWalletVault`, the required waddrmgr operation, and the
  one modern-kvdb account-creation path that currently derives and encrypts
  while its walletdb write transaction is open.
- Prepare and validate both old credentials before changing persistent or live
  key state.
- Hold one manager-internal operation guard across preparation, database commit,
  and live-key installation so lock, unlock, encryption, or another rotation
  cannot observe or overwrite an intermediate phase.
- Establish one lock order: acquire the operation guard before every relevant
  `walletdb.View` or `walletdb.Update`, including lock, unlock, encryption, and
  rotation paths. Never acquire the guard from inside an open walletdb
  transaction.
- For modern-kvdb account creation, read the expected account cursor under the
  guard, close the read transaction, derive and encrypt into a prepared plan,
  then use one short write transaction to compare-and-set the cursor and insert
  ciphertext. No encryption or guard acquisition runs inside that write.
- Apply the sequential invalid-child semantics to that prepared path:
  atomically consume invalid account and branch derivations and persist terminal
  exhaustion. Exact account creation remains permanently unsupported on the
  maintained kvdb Manager under [Task 343][t343] and has no branch in this task.
- Publish new in-memory keys only after the database transaction commits.
- Preserve the pre-rotation lock state. A locked manager remains locked with no
  decrypted private or script keys, while retaining the public crypto material
  that waddrmgr requires while locked. An unlocked manager remains usable only
  with the new credentials after a successful rotation.
- Treat a bbolt commit error as potentially ambiguous. Reopen or reread the
  exact public and private verifier facts, install the matching old or new live
  keys, and return a wallet-owned indeterminate error if neither state can be
  proven.
- Zero every prepared master key and derived crypto key on abort and after
  successful installation.
- Deep-copy every prepared key into manager-owned live storage before zeroing
  plan-owned buffers. Installation transfers values, never ownership of the
  buffers that the plan destroys.
- Do not change deprecated RPC behavior or add a new kvdb format.

## Suggested fix

Add an operation-specific waddrmgr rotation plan that acquires an internal guard
and derives and validates both halves without mutating the manager. Apply its
database writes in one `walletdb.Update`, install prepared in-memory keys only
after a successful or reconciled-new commit, then zero and release the plan. A
destroy path zeroes and releases it after reconciled-old failure. Audit and
update every related path to use guard-before-transaction ordering. Record the
initial lock state in the plan and install deep copies only when that state
requires live decrypted keys. Give account creation a narrow prepared account
plan with expected-cursor CAS so it follows the same guard-before-walletdb
order.

## Verification

Injected failure before either write, between writes, and at commit leaves disk
and live manager state on the same proven credentials; success switches both to
the new credentials together. An ambiguous commit is reconciled as old, new, or
typed indeterminate before prepared keys are installed or destroyed. Concurrent
lock, unlock, encryption, and rotation cannot enter between phases, lock-order
tests do not deadlock, and plan buffers are zeroed on every exit. Locked and
unlocked rotations preserve their state, and zeroing the plan cannot corrupt
manager-owned keys. Locked rotation retains usable public crypto keys but no
private or script keys. A barrier-controlled account creation against rotation
cannot deadlock, observe mixed credentials, or retain a partial account.
Invalid-child and terminal-exhaustion fixtures preserve the cursor policy while
rotation holds the same operation guard. No exact-account request or test is
added to the modern kvdb path.

## Definition of done

- Legacy public and private credentials rotate as one proven disk-and-memory
  result while preserving the manager's initial lock state.
- Account creation, lock, unlock, encryption, and rotation share one explicit
  guard-before-walletdb order without plaintext retention or deadlock.
- The PR keeps the legacy dual-encryption format and adds no exact-account or
  deprecated RPC behavior.

[t343]: 343-exact-account-creation.md
