# Task 329 — make wallet and account creation atomic

Component: **Manager and AccountManager**.

Dependencies: **[Task 332][t332], [`337.sql-selector`][t337], and
[`345.sql-discovery`][t345]**. [Task 341][t341] and [Task 340][t340] are
transitive prerequisites of those two milestones.

## Rationale

Wallet rows and secrets are already created transactionally, but default and
initial accounts are added afterwards. A failure can therefore leave a wallet
without the accounts its create request promised.

## Details

- Cover SQLite, PostgreSQL, and the canonical kvdb create transaction.
- Define one normalized genesis plan before mutation. Every private-root mode,
  including generated seed, imported seed, and imported XPrv, creates `default`
  in every canonical `waddrmgr.DefaultKeyScopes` scope. A shell creates exactly
  its requested initial accounts and no default account.
- Accept an imported XPrv as a wallet root only when its depth, parent
  fingerprint, and child index are all zero and its version belongs to the
  selected network. Reject every non-root or wrong-network key before Vault or
  Store access.
- Validate every initial account as one identity before mutation: its XPub
  version and resulting address encoding must match the selected network, and
  its depth, address type, and declared scope must match the scope's single
  address schema. Treat purpose and coin type as persisted or requested scope
  identity; do not rewrite or validate the scope coin type against
  `ChainParams.HDCoinType`. In particular, retain the canonical coin-type-zero
  `waddrmgr.DefaultKeyScopes` on lnd test networks. Do not discard a conflicting
  declared scope and persist an inferred replacement.
- Use [Task 341][t341]'s optional fingerprint representation for initial accounts so
  SQL preserves absent and present-zero distinctly. Legacy kvdb persists only
  one four-byte fingerprint, so reject an absent initial fingerprint before
  mutation there; a stored zero is consequently known to mean present-zero.
  Reconcile each backend only against the fingerprint states it can represent.
- Normalize each initial account XPub payload once and compare it wallet-wide
  before Vault preparation. Reject equal payloads whenever the accounts'
  effective external or internal branch schemas overlap, even across scopes;
  names, version bytes, optional metadata, and zero address counters do not make
  colliding child-key identities safe.
- Persist `watch-chain` for canonical private-root default accounts. Require
  every shell initial account to declare its immutable [Task 345][t345] discovery mode
  explicitly and include it in genesis reconciliation.
- Persist each scope's checked `next_account_number` in the same plan. A
  private-root scope containing canonical account zero starts at one. Shell
  initial XPub accounts are numberless and never advance that cursor; this task
  has no public initial-account number to honor and must not invent one. A
  numbered initial-account shape waits for [Task 343][t343].
- Reject an initial account whose address-schema override differs from another
  account in the same scope. The SQL model has one scope schema and cannot
  represent conflicting per-account overrides truthfully.
- Treat kvdb's reserved imported buckets as backend infrastructure, not extra
  public accounts. Reject an XPub-root shape that cannot name truthful initial
  accounts rather than succeeding with an empty backend-specific wallet.
- Require [Task 337][t337]'s wallet-owned selector before admitting numberless shell
  accounts on SQL. The maintained kvdb backend may accept shell accounts only
  if [Task 337][t337]'s modern-kvdb representation lands first; otherwise return a
  stable unsupported error before mutation rather than fabricating a BIP44
  account number.
- Keep SQL key derivation and encryption outside database write transactions.
  The retiring kvdb path may keep canonical `waddrmgr.Create` inside its one
  transaction; do not split waddrmgr solely for this task.
- Keep only ciphertext and public derivation facts in the genesis plan. Bound
  plaintext account-key lifetime inside the Vault preparation call and zero it
  on success, validation failure, write failure, and panic.
- Keep generated seed bytes in wipeable storage and zero them on every exit.
  Do not retain a plaintext root serialization in the plan. If the upstream HD
  key API forces an immutable string temporarily, confine it to the Vault call,
  retain no reference, and document and test that bounded exposure.
- Treat an internally derived master `ExtendedKey` as task-owned and arrange
  its zeroing immediately after derivation so every later exit clears it. For an
  imported XPrv, validate the caller's key first, make a separate owned working
  clone, zero only that clone, and never mutate or zero the caller-owned key.
- Treat SQL and kvdb commit errors as potentially ambiguous. Reopen and
  reconcile the complete wallet, secret, sync, scope, and account genesis
  facts. Return success only for an exact committed plan, ordinary failure only
  for no artifact, and a wallet-owned indeterminate error for any other state.
- Publish the Wallet in `Manager.wallets` only after the complete genesis
  transaction commits or reconciles as committed and runtime assembly
  succeeds. A failed initial-account plan must leave no cached Wallet.
- Do not add create manifests, replay markers, or Store callbacks.
- Keep this independent of duplicate-create policy in [Task 330][t330].

## Potential fix

Have the Vault prepare a mode-specific ciphertext-only genesis plan and destroy
its plaintext working state before the SQL write. Extend each backend's existing
wallet-create transaction to insert the wallet, secrets, sync state, scopes, and
accounts and checked cursors together. Commit through a small backend-local
operation rather than widening `db.Store` with a derivation callback. Reconcile
an ambiguous SQL or bbolt commit from those exact genesis facts rather than
adding a create manifest.

## Done when

A definite fault during any account insert leaves no wallet artifact on either
SQL dialect or kvdb, and successful creates expose the same normalized public
genesis account set for requests supported by both backends. SQL distinguishes
absent from present-zero fingerprints; kvdb rejects absent before mutation and
round-trips present-zero. Ambiguous SQL and bbolt commit tests prove exact-
commit, no-artifact, and indeterminate outcomes. Focused fault injection proves
no seed, serialized root, or account working key is retained after any exit.
Reopen proves every private-root scope cursor is one, shell accounts leave the
cursor unchanged, and conflicting account schemas fail before mutation.
Non-root and wrong-network XPrvs fail before dependency access. This task does
not add cancellation to the context-free public `Manager.Create` API; [Task 358][t358]
owns that replacement. Declared-scope, key-depth, address-type, schema,
discovery-mode, wrong-network initial-XPub, optional-fingerprint, and duplicate-
XPub fixtures fail or round-trip as appropriate before mutation. No Wallet is
published after a failed or indeterminate genesis. Generated and imported-seed
master keys and the owned clone of an imported XPrv are zeroed on every exit;
the caller's imported XPrv remains unchanged and usable. A zero-counter
BIP49Plus/BIP84 initial-account fixture is rejected before creation when its
normalized payload would make an overlapping branch derive the same scripts.
Standard coin-type-zero scopes on testnet, simnet, and signet round-trip while
an XPub or address encoding for another network fails before mutation.

[t330]: 330-duplicate-wallet-create.md
[t332]: 332-watch-only-script-policy.md
[t337]: 337-imported-xpub-derivation.md
[t340]: 340-durable-database-identity.md
[t341]: 341-public-role-contract.md
[t343]: 343-exact-account-creation.md
[t345]: 345-durable-watch-registration.md
[t358]: 358-contextual-manager-operations.md
