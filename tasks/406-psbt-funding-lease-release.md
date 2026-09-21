# Task 406 — clean up failed PSBT funding leases

Component: **PsbtManager private failure cleanup and UTXO Store**.

Type: **Gap fix**.

Dependencies: **[Task 405][t405]**.

## Rationale

`FundPsbt` must not return a packet after a later watch or assembly failure,
but its newly reserved inputs must not remain leased or release a replacement
owner.

## Details

- Keep exact-set cleanup private to failures before `FundPsbt` returns. It
  receives the selected outpoints and expected `FundIntent.LeaseID` from the
  funding attempt; do not add a public PsbtManager cleanup method or result.
- Use a bounded wallet-owned context after caller cancellation and release
  only leases still owned by the expected ID. Finish accepted cleanup before
  Store closure.
- Attempt cleanup for the whole selected set, including after a release
  fails or reservation returned an ambiguous commit. Preserve replacement
  owners, the original funding error, and cleanup failures. Unknown cleanup
  matches Task 467's `ErrIndeterminateCommit`; do not claim success.
  Add no ambiguous-outcome readback or recovery loop.
- After successful `FundPsbt`, callers use the packet outpoints, their lease
  ID, and existing idempotent `ReleaseOutput` rather than this private path.
- Lease expiry is the fallback for unreleased outputs, not a substitute for
  immediate cleanup.

Exclusions: input selection, lease-ID generation, watched-change delivery,
public batch cleanup, new per-outpoint lease semantics, durable cleanup
daemons, and generic replay state.

## Suggested fix

Reuse existing ownership-checked `ReleaseOutput` semantics: SQL
`ReleaseOutputWithOps` and the kvdb release path already protect a replacement
owner. Bounded per-outpoint releases may clean up the attempted set; no new
atomic batch-delete API or cleanup state machine is required.

## Verification

A post-admission failure releases the attempted set even after caller
cancellation. Ambiguous reservation still attempts every selected outpoint.
A release failure does not skip remaining outpoints, the original error and
unknown cleanup remain observable, and a replacement owner survives.
Cleanup finishes before Store closure without readback or recovery loops.
Reuse existing `ReleaseOutput` ownership tests rather than duplicating them.
[Task 439][t439] owns shared public integration coverage for definitive
pre-return cleanup and the final caller-visible funding contract; this task
retains focused private cleanup-error coverage.

## Definition of done

Every failure before `FundPsbt` returns attempts cleanup of all selected
leases and either removes the attempt's ownership or reports the remaining
failure honestly, using shared indeterminacy when the outcome is unknown.
Replacement owners remain untouched. No new public result, cleanup method,
or PSBT-specific error is added.

[t405]: 405-psbt-funding-lease-admission.md
[t467]: ../archive/tasks/467-indeterminate-commit-identity.md
[t439]: 439-psbt-funding-itests.md
