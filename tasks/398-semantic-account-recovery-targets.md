# Task 398 — accept semantic account recovery targets

Component: **Controller and Syncer recovery account selection**.

Dependencies: **[Task 341][t341], [Task 345][t345], [Task 397][t397], and
[Task 399][t399]**.

## Rationale

The Controller exposes backend-shaped numeric targets and cannot
target an SQL numberless imported-XPub account. The runtime target contract is
independent of derivation budgeting and zero-lookahead Store queries.

## Details

- Replace `[]waddrmgr.AccountScope` at the maintained public Controller
  boundary with Task 341's wallet-owned scope-plus-exactly-one-name-or-number
  selector.
- Validate neither/both-set, duplicates, not-found, and malformed scope before
  starting a scan.
- Resolve names and optional numbers to internal durable account identities
  once, preserving rename-safe horizon updates without exposing those IDs.
- Reject key-only targets and accountless raw-import aliases.
- For an ordinary full positive-lookahead recovery with no explicit targets,
  enumerate derivation horizons only for persisted `watch-chain` accounts.
  Apply the mode filter before any horizon derivation or watch construction so
  a key-only account is never an implicit recovery target.
- Resolve semantic targets into Task 399's completed internal account-identity
  input, then use its exact persisted address/outpoint set for both zero and
  positive lookahead. The public boundary must not land with the old
  wallet-wide persisted-target loader still reachable.
- Preserve the current modern-kvdb numeric/account-name adapter internally.
  This task does not require Task 361's public derivation API and adds no new
  kvdb storage format. If kvdb retires first, remove only that adapter clause.
- Do not add a second persisted-target loader or widen Task 399's exact sets.

## Suggested fix

Translate the public semantic selector into one internal
scan target at request admission, then carry that identity through horizon
loading and persistence. Route default full-scan account enumeration through
the same discovery-mode filter before deriving any horizon, and route targeted
persisted state through Task 399 before exposing the public selector.

## Verification

SQL numbered and numberless-name targets, kvdb ordinary
numeric targets, rename, collision, neither/both, duplicate, key-only, and
accountless cases prove exact admission without backend IDs. A full positive
lookahead scan with mixed watch-chain and key-only accounts derives and watches
only the watch-chain horizons.

## Definition of done

Maintained targeted recovery accepts one wallet-owned
semantic account contract and can name SQL numberless imported-XPub accounts
without depending on Task 361 or exposing Store identity. Ordinary full
positive-lookahead recovery never derives or watches a key-only account.

[t341]: ../archive/tasks/341-public-role-contract.md
[t345]: 345-durable-watch-registration.md
[t397]: 397-semantic-recovery-targets-adr.md
[t399]: 399-zero-lookahead-persisted-targets.md
