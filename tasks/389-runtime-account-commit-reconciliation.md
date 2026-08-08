# Task 389 — reconcile ambiguous runtime account commits

Component: **AccountManager and SQL AccountStore account result reader**.

Dependencies: **[Task 343][t343]**.

## Rationale

The SQL runtime exposes indeterminate commit responses for every
write, not only Manager genesis. Retrying an ordinary import or an exact or
sequential root-derived account from the transport error can duplicate an
account or advance a cursor twice.

## Details

- Handle `runtime.ErrAmbiguousTxCommit` for ordinary SQL `ImportAccount`, the
  retained Task 343 account-creation operation, and no later account shape.
- Re-read only their existing scope, account identity, XPub, fingerprint,
  secret presence, discovery mode, skipped/exhausted state, and cursor facts.
  Keep the result reader extensible without naming or comparing custom-scope or
  path-bound facts that do not exist yet.
- Return committed success only for an exact match, ordinary failure only when
  the attempted operation left no fact, and a wallet-owned indeterminate error
  for partial/conflicting state.
- Never retry, add a manifest, or generalize this into a Store-wide framework.
- Keep Task 366 exclusively responsible for Manager SQL genesis reconciliation.

## Suggested fix

Add an operation-specific account result reader and exact
comparison against the prepared attempt, then translate its committed, absent,
or conflicting outcome once at AccountManager.

## Verification

Fault injection on both dialects covers ordinary imports and Task 343
root-derived creation for exact commit, absent commit, and conflicts in every
existing fact. Only exact state returns one account; no case derives, inserts,
or advances again.

## Definition of done

Every ambiguous ordinary-import or Task 343 SQL account result is classified
without retry or duplicate mutation, and no later account shape, genesis,
allocation, or generic reconciliation mechanism is added.

[t343]: 343-exact-account-creation.md
