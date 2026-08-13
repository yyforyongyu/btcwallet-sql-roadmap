# Task 401 — add an idempotent live-tip-only chain watch

Component: **chain interface and Neutrino/btcd/bitcoind adapters**.

Type: **Gap fix**.

Dependencies: **[Task 400][t400]**.

## Rationale

`NotifyReceived` is not a safe live-only primitive. Neutrino can
start a birthday-based rescan or widen the active historical rescan.

## Details

- Add one context-aware, idempotent chain operation that adds supported address
  watches only at the live-tip boundary and scans no historical block.
- Implement equivalent behavior for btcd, bitcoind, polling, mocks, and
  Neutrino without changing legacy `NotifyReceived` semantics for deprecated
  callers.
- During an active Neutrino historical scan, isolate or defer the new watch
  until the live-tip boundary. Never mutate that scan's start height, target
  set, progress, or completion.
- Keep Store generations, replay, pending results, and recovery out of scope.

## Suggested fix

Expose a narrow `AddLiveWatch`-style chain method and give
Neutrino a separate live watch set or deferred update path instead of calling
the active rescan's `Update` blindly.

## Verification

Adapter conformance proves idempotence and no history; a
barrier-controlled Neutrino scan proves live registration neither widens nor
restarts it and becomes active at the documented tip boundary.

## Definition of done

Every maintained chain backend can add one live-only
watch idempotently, and Neutrino preserves active historical scan identity.

[t400]: 400-live-watch-semantics-adr.md
