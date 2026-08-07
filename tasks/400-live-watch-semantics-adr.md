# Task 400 — separate live watches from historical recovery

Component: **ADR 0004 and ADR 0005 synchronization contract**.

Dependencies: **None**.

## Rationale

ADR 0004 says imports trigger targeted historical scans, while
ADR 0005 says imports are database-only and rescans are explicit. Durable live
watch delivery cannot be implemented against contradictory accepted records.

## Details

- Remove import-triggered historical recovery from ADR 0004 and preserve ADR
  0005's synchronous database-only import return.
- Define asynchronous live-tip watch maintenance as distinct from explicit
  full or semantic-account historical recovery.
- State that live registration never changes a historical scan's start height,
  target set, or birthday.
- State that accountless raw imports require an explicit full-wallet historical
  scan; account-scoped history uses the separately accepted semantic target
  contract.
- Define accepted/pending/ready terminology for scripts without designing a
  generic notification outbox or durable rescan job.

## Suggested fix

Amend ADR 0004 and ADR 0005 together with one timeline for
database commit, durable live obligation, asynchronous live-tip delivery, and
optional explicit history recovery.

## Verification

Walk allocation, raw import, restart, live-tip update, active
historical scan, and explicit recovery examples through both records and prove
they no longer prescribe incompatible behavior.

## Definition of done

Accepted ADR text gives live watch delivery and
historical recovery disjoint triggers and state effects, with no implementation
or generic delivery design.
