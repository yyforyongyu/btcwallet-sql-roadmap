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
  only leases still owned by the expected ID.
- Attempt cleanup for the whole selected set, including after a release
  fails. Preserve replacement owners and report partial failure honestly.
  Reconcile ambiguous outcomes only where required to determine whether the
  attempt still owns leases; unresolved state matches Task 467's shared
  `ErrIndeterminateCommit`. Do not claim successful cleanup while ownership
  remains unknown.
- After successful `FundPsbt`, callers use the packet outpoints, their lease
  ID, and existing idempotent `ReleaseOutput` rather than this private path.

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
cancellation. A release failure does not skip remaining outpoints, partial or
unknown cleanup is reported honestly, and a replacement owner survives.
Reuse existing `ReleaseOutput` ownership tests rather than duplicating them.
[Task 439][t439] owns shared public integration coverage for definitive
pre-return cleanup and the final caller-visible funding contract; this task
retains focused private classification coverage.

## Definition of done

Every failure before `FundPsbt` returns attempts cleanup of all selected
leases and either removes the attempt's ownership or reports the remaining
failure honestly, using shared indeterminacy when the outcome is unknown.
Replacement owners remain untouched. No new public result, cleanup method,
or PSBT-specific error is added.

[t405]: 405-psbt-funding-lease-admission.md
[t467]: ../archive/tasks/467-indeterminate-commit-identity.md
[t439]: 439-psbt-funding-itests.md
