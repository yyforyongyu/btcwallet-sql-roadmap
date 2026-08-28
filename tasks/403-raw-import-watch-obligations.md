# Task 403 — persist raw imports with live-watch obligations

Component: **AddressManager SQL raw-import mutation**.

Type: **Bug fix**.

Dependencies: **[Task 332][t332], [Task 364][t364], and [Task 402][t402]**.

## Rationale

Maintained public-key and Taproot import paths currently commit
the address and call `NotifyReceived` afterward, leaving no durable retry fact.

## Details

- For every maintained SQL accountless raw-import path, commit the exact import
  and one ordered watch obligation in the same method-specific transaction.
- Keep secret encryption outside the Store transaction and consume Task 332's
  accepted import policy without reopening it.
- Remove direct `NotifyReceived` calls from the SQL path. Return after database
  commit with the accepted/pending semantics from the ADR; never start a
  historical scan.
- Reconcile an ambiguous import commit from exact script, secret provenance,
  generation, obligation, and delivery facts.
- Let Task 402 deliver asynchronously. An accountless import requires a
  separate explicit full-wallet recovery request for historical discovery.
- Keep modern-kvdb import behavior outside this SQL mutation without adding a
  durable kvdb format; Task 353 may remove that backend path later.

## Suggested fix

Extend the existing SQL `NewImportedAddress` method-specific
transaction with retained Task 364 obligation insertion and exact result
re-read, then delete Wallet-side chain notification on the SQL branch.

## Verification

SQLite and PostgreSQL cover commit-before-chain ordering, ambiguous
exact/absent/conflicting outcomes, chain failure, fresh-Manager replay, and
duplicate import. Each accepted public-key and Taproot raw import persists one
obligation that Task 402 registers only at the live tip, without starting a
historical scan. kvdb on-disk behavior is unchanged.

## Definition of done

Every accepted maintained SQL raw import has one
durable ordered live-watch obligation and no direct or automatic historical
chain operation.

[t332]: ../archive/tasks/332-watch-only-script-policy.md
[t364]: 364-durable-watch-registration.md
[t402]: 402-replay-live-watch-obligations.md
