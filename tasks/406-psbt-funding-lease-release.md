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
- Reconcile cleanup as all gone, all still owned by the expected ID, replaced
  by another owner, partial, or unavailable. Never release a replacement
  owner; wrap unresolved state with Task 467's shared
  `ErrIndeterminateCommit` and keep operation-specific states private.
- After successful `FundPsbt`, callers use the packet outpoints, their lease
  ID, and existing idempotent `ReleaseOutput` rather than this private path.

Exclusions: input selection, lease-ID generation, watched-change delivery,
public batch cleanup, new per-outpoint lease semantics, durable cleanup
daemons, and generic replay state.

## Suggested fix

Add only the private exact-set delete and matching state read needed by the
existing `FundPsbt` failure path, then translate unresolved outcomes through
the shared public identity.

## Verification

Exact-set cleanup is atomic; cancellation still invokes its bounded context;
all-gone/all-owned/replaced/partial/unavailable outcomes are classified; and
cleanup never affects a lease owned by another ID. Public success cleanup uses
focused `ReleaseOutput` coverage instead of a new method.
[Task 439][t439] owns shared public integration coverage for definitive
pre-return cleanup and the final caller-visible funding contract; this task
retains focused private classification coverage.

## Definition of done

Every failure before `FundPsbt` returns either removes exactly the attempt's
leases or reports shared indeterminacy without releasing a replacement. No new
public result, cleanup method, or PSBT-specific error is added.

[t405]: 405-psbt-funding-lease-admission.md
[t467]: 467-indeterminate-commit-identity.md
[t439]: 439-psbt-funding-itests.md
