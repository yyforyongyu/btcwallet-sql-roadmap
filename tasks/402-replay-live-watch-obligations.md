# Task 402 — replay Wallet live-watch obligations

Component: **Wallet live-watch delivery and replay**.

Type: **Gap fix**.

Dependencies: **[Task 334][t334], [Task 364][t364], and [Task 401][t401]**.

## Rationale

Persisting a watch obligation does not make an address safe to use. After a
restart the chain backend has forgotten prior registrations, and a failed live
registration can leave durable work incomplete. Wallet must rebuild and retry
those watches without exposing delivery bookkeeping to callers.

## Details

- At Wallet startup, load a coherent persisted watch snapshot and register it
  at the live tip before delivering later obligations in durable order.
- Re-register completed rows after process restart because backend watch state
  is not durable. Mark completion only after successful chain registration.
- Retry incomplete obligations while Wallet is running. Failure or
  cancellation must leave enough durable state for a later retry or restart.
- An address-allocation API may wait for its own registration before returning
  a usable address. Raw-import producers retain their separately defined
  return contract.
- Wallet stop must cancel and join its delivery work before Store closure.
- Preserve the same behavior before and after Task 423 changes the chain-event
  owner. Do not create duplicate replay state or require a new implementation
  type.
- Keep delivery state private. Exclude producer mutations, historical scans,
  `NoChainSync` children, canonical-frontier movement, and a general
  notification framework.

## Suggested fix

Extend the existing Wallet synchronization path to replay Task 364's durable
obligations through Task 401's live-tip registration. The worker and retry
mechanism are implementation choices, provided stop owns their lifetime.

## Verification

- Prove startup snapshot registration precedes later generations and address
  results are not usable before their required live registration.
- Cover duplicates, transient failure, cancellation, and restart without lost
  or reordered obligations.
- Block registration during stop and prove delivery is joined before Store
  closure; a fresh startup completes the retained obligation.
- Prove replay cannot start history or move the canonical frontier.

## Definition of done

Wallet provides ordered, restart-safe live-watch delivery with private durable
state and a complete shutdown path, without a new public API or notification
framework.

[t334]: ../archive/tasks/334-shared-chain-state.md
[t364]: 364-durable-watch-registration.md
[t401]: ../archive/tasks/401-live-tip-chain-watch.md
