# Task 359 — revalidate database identity design

Component: **Manager and SQL Store design**.

Dependencies: **None**.

## Rationale

[Task 340](340-durable-database-identity.md) must reject foreign or
wrong-network databases before mutation, but its identity tuple and first-open
order need a current design review against both SQL Store branches before code
can safely depend on them.

## Details

- Recheck the minimal schema-family, generation, genesis, network-magic, and
  behavior-parameter identity tuple against current SQLite and PostgreSQL
  initialization and migration paths.
- Confirm that custom signet requires the trusted full challenge digest in
  addition to truncated network magic.
- Define the read-only preflight, empty-unmarked adoption, metadata bootstrap,
  and ordinary forward-migration order.
- Identify the database-native exclusion that serializes concurrent first
  open without adding a process-local correctness lock.
- Confirm that populated unmarked, foreign, dirty, newer, and mismatched
  databases remain non-mutating rejection cases.
- Keep archived
  [Task 1000](../archive/tasks/1000-complete-btcwallet-sql-role-readiness.md)
  lock, manifest, and generic recovery machinery out of scope.
- Produce only the accepted design record. Store implementation, schema
  changes, generated output, and migration tests belong to
  [Task 340](340-durable-database-identity.md).

## Suggested fix

Inspect the current dialect open and migration paths, then record the smallest
sufficient identity tuple and exact preflight/adoption sequence in the owning
design documentation. Include the explicit signet input and native first-open
exclusion without prescribing generic recovery infrastructure.

## Verification

- Trace SQLite and PostgreSQL open paths to prove the accepted sequence can
  inspect identity before ordinary migrations or wallet queries mutate data.
- Exercise the design against empty, matching, foreign, populated-unmarked,
  dirty, newer, wrong-network, and equal-magic/different-signet scenarios.
- Confirm every tuple field changes maintained Wallet behavior and that no
  implementation-only mechanism is admitted by the record.

## Definition of done

- An accepted design record fixes the minimal identity tuple, explicit signet
  input, preflight/adoption order, and native first-open exclusion.
- [Task 340](340-durable-database-identity.md) can implement the record without
  reopening a product or storage-order decision.
- The PR contains no Store implementation, schema migration, generated output,
  generic lock, create manifest, or recovery framework.
