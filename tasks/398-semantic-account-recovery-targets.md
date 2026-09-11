# Task 398 — accept semantic account recovery targets

Component: **Controller and Syncer recovery account selection**.

Type: **Gap fix**.

Dependencies: **[Task 341][t341], [Task 345][t345], [Task 397][t397],
[Task 399][t399], and [Task 452][t452]**.

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
- Preserve Task 452's pre-derivation rejection of explicit `NoChainSync`
  targets and its `wallet.ErrNoChainSyncRecoveryTarget` identity when replacing
  the numeric selector, and reject accountless raw-import aliases.
- Preserve Task 452's default full positive-lookahead `NoChainSync` filter.
  Semantic target admission must not make excluded accounts implicit or
  explicit recovery targets again.
- Resolve semantic targets into Task 399's completed internal account-identity
  input, then use its exact persisted address/outpoint set for both zero and
  positive lookahead. The public boundary must not land with the old
  wallet-wide persisted-target loader still reachable.
- Resolve maintained modern-kvdb numeric and account-name targets behind the
  final semantic public request. Remove the old public target shape and retain
  no old-shape adapter. This task does not require Task 361's public
  derivation API and adds no new kvdb storage format. If kvdb retires first,
  remove only this backend-resolution clause.
- Do not add a second persisted-target loader or widen Task 399's exact sets.

## Suggested fix

Translate the public semantic selector into one internal
scan target at request admission, then carry that identity through horizon
loading and persistence. Preserve Task 452's full-scan filter and explicit-
target rejection, then route targeted persisted state through Task 399 before
exposing the public selector.

## Verification

SQL numbered and numberless-name targets, kvdb ordinary
numeric targets, rename, collision, neither/both, duplicate, `NoChainSync`, and
accountless cases prove exact admission without backend IDs. Regression
coverage proves semantic admission preserves both Task 452 recovery guards and
its `errors.Is(err, wallet.ErrNoChainSyncRecoveryTarget)` result.

## Definition of done

Maintained targeted recovery accepts one wallet-owned
semantic account contract and can name SQL numberless imported-XPub accounts
without depending on Task 361 or exposing Store identity. The public selector
preserves Task 452's full-scan filtering, explicit-target rejection, and stable
wallet-owned error identity.

[t341]: ../archive/tasks/341-public-role-contract.md
[t345]: ../archive/tasks/345-durable-watch-registration.md
[t397]: 397-semantic-recovery-targets-adr.md
[t399]: 399-zero-lookahead-persisted-targets.md
[t452]: 452-key-only-recovery-derivation.md
