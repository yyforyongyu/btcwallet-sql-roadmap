# Task 406 — release PSBT funding leases by exact token

Component: **UTXO Store and PsbtManager cleanup**.

Dependencies: **[Task 405][t405]**.

## Rationale

Expose one narrow atomic release operation for the
complete operation-owned set.

## Details

It accepts the typed token/result rather than
caller-reconstructed outpoints, uses a bounded wallet-owned context after
caller cancellation, and reconciles ambiguous cleanup as all gone, all still
owned by that token, replaced by another token, or indeterminate. It never
releases a replacement token.

Exclusions: input selection, token generation, watched-change delivery,
per-outpoint public lease semantics, durable cleanup daemons, and generic
replay state.

## Suggested fix

Add one Store delete-by-exact-token-and-set operation and a
matching set-state read, then expose a PsbtManager cleanup method that owns its
bounded context and outcome translation.

## Verification

Exact-set release is atomic; cancellation still invokes the
bounded cleanup; all-gone/all-owned/replaced/partial/unavailable outcomes are
classified; delayed old-token cleanup cannot affect a later lease.

## Definition of done

Callers holding only the wallet-owned funding result
can atomically release exactly its set, and no cleanup outcome is reported as
definite unless reconciled.

[t405]: 405-psbt-funding-lease-admission.md
