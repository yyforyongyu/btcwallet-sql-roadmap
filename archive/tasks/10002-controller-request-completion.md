# Task 10002 — controller request-completion tracking

Status: **retired as duplicate reference.** [[Task 1000]] already describes this
protocol under `BWR-001` — caller context carried through Controller requests, a
`queued -> claimed -> finished` state machine, and definitive post-claim results
(`:753-777`). This file is the extracted version of the same work, so keeping it
as a separate owner would mean two plans for one invariant. Read it for the
defect analysis; do not schedule it. 1000 stays untouched.

Originally created as: Created so that [[Task 305]]'s acknowledged
systemic debt has a durable owner instead of living as prose in a commit
message. Both GPT-5.6 and Fable 5 raised this independently in round 2: a
follow-up recorded only in narrative is not a work item and cannot be tracked to
completion.

## The Defect

Every Controller operation routed through `sendReq`/`waitForResp` can report
failure while its handler completes successfully.

Verified at `impl-secret-store` (`7bf664f83`): `sendReq` (`controller.go:830`)
hands the request to the main loop; the handler then runs under
`w.lifetimeCtx` — deliberately **not** the caller's context — and commits. But
`waitForResp` (`:845`) can return `ctx.Err()` or `ErrWalletShuttingDown` while
that handler is still running. The caller is told the operation failed; the
mutation lands anyway.

This is not specific to passphrase rotation. It is the shape of the shared
request plumbing, so it applies to every operation that uses it.

## Why Task 305 Fixes Only One Path

Task 305 gives the passphrase path its own wait that does not abandon an
accepted request, and deliberately does **not** change `waitForResp` globally.
Two reasons, both still valid:

- passphrase rotation is the one operation where an ambiguous success means the
  caller may lose access to a wallet whose credentials did change;
- `waitForResp` is shared by every Controller operation, so changing it inside a
  passphrase commit would be an unreviewable blast radius.

That makes 305's narrow fix correct **and** makes it an exception. This task
exists so the exception is removed rather than becoming the pattern.

## Direction

Request-completion tracking at the **protocol** level: once the handler loop
accepts a request, the caller learns its definitive outcome regardless of caller
cancellation — uniformly, for all operations.

**One contract, chosen: bounded completion plus a queryable result.** An earlier
draft of this section promised definitive completion *and* preferred "a bounded
wait plus a typed ambiguous-outcome error", which are not the same contract — a
typed ambiguous error is honest about not knowing, but it still leaves the
caller without the outcome, and it would **regress Task 305's passphrase
guarantee** the moment the special path is deleted. So:

- **accepted handlers have bounded completion**, and the shared wait returns
  their definitive result within that bound;
- if the bound is exceeded, the call returns an **operation ID whose result
  stays queryable** for a defined lifetime — not a bare ambiguous error;
- **restart durability is deliberately out of scope**, so state the consequence
  explicitly: if the Controller stops before an in-memory result is queried,
  that result is lost, and the operation's effect must be discoverable by
  reading wallet state instead. Any consumer that genuinely needs to learn an
  outcome across a restart is what would justify a persistence-backed registry
  later.

The one thing this task may not do is promise definitive completion, remove the
passphrase exception, and still permit an unqueryable ambiguous result.

Design constraints worth carrying in:

- **Acceptance is the commitment point.** Cancellation before acceptance may
  abort; after acceptance the caller must learn the true outcome. Task 305
  documents this boundary for the passphrase path; this task generalizes it.
- **The invariants 305 pins are what make it safe** and must be preserved or
  explicitly replaced: `requestChan` is unbuffered so acceptance is a rendezvous
  (`manager.go:349` at #1272 / `:1490` at stack tip), and every handler path
  writes a **buffered** `resp` (`controller.go:682/689/698` at #1272 /
  `:1159/1166/1175` at tip). If a buffered `requestChan` is ever introduced,
  accepted-but-unprocessed requests become possible at shutdown and the design
  needs a loop-exited escape plus a final non-blocking `resp` read.
- **Do not regress shutdown latency.** Waiting for an accepted request must not
  let a stuck handler hold shutdown open indefinitely; bound it or make the
  outcome retrievable after restart.

## Scope

- generalize the accepted-request guarantee across every `sendReq`/`waitForResp`
  caller;
- remove the passphrase-specific wait Task 305 introduces, so there is one
  mechanism rather than two;
- document the cancellation boundary once, on the shared plumbing, rather than
  per method.

## Prerequisites

Task 305 lands first — this task deletes the special case 305 introduces, so
doing them in the other order would mean writing the exception twice.

## Definition Of Done

1. Accepted requests report their true outcome for **every** Controller
   operation, not just passphrase rotation — within the bound, or via a
   queryable operation ID afterwards. No path returns an ambiguous result the
   caller cannot resolve.
2. Task 305's passphrase-specific wait is deleted, and its exported cancellation
   note is replaced by one statement on the shared plumbing.
3. A test per operation class proves cancellation after an explicit acceptance
   latch still returns the definitive result — the acceptance latch matters,
   since cancelling before acceptance passes against the buggy code too.
4. Shutdown latency is bounded, with a test covering a slow handler at shutdown.
5. The `requestChan`/`resp` buffering invariants are either preserved and
   asserted, or explicitly replaced with the escape path described above.
