# Task 397 — define semantic-account recovery targets

Component: **ADR 0004 targeted-recovery contract**.

Dependencies: **[Task 341][t341] and [Task 400][t400]**.

## Rationale

The public numeric `waddrmgr.AccountScope` cannot name a
numberless imported-XPub account, and ADR 0004 still describes individual
address/script import targets rather than explicit semantic accounts.

## Details

- Amend ADR 0004 to make an explicit targeted recovery request select semantic
  accounts by scope plus exactly one name or optional BIP44 number.
- Reject SQL row IDs and kvdb adapter IDs as public identity.
- Define target deduplication, not-found behavior, key-only exclusion, and
  accountless raw-import exclusion.
- State that accountless raw imports are recovered only by an explicit full
  wallet scan.
- Keep live-watch semantics delegated to the prerequisite ADR and leave
  durable rescan jobs out of scope.

## Suggested fix

Add one semantic-target section and examples for numbered,
numberless imported-XPub, key-only, renamed, and accountless identities.

## Verification

Compare the record with Task 341's selector and current SQL
and kvdb adapter identities; prove every supported target is nameable without
backend IDs and every excluded shape has an explicit route.

## Definition of done

Accepted ADR 0004 text fixes one backend-neutral
semantic-account recovery target and no implementation or live-watch behavior.

[t341]: ../archive/tasks/341-public-role-contract.md
[t400]: 400-live-watch-semantics-adr.md
