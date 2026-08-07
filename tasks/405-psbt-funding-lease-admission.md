# Task 405 — reserve PSBT funding inputs atomically

Component: **PsbtManager and UTXO Store funding admission**.

Dependencies: **[Task 346][t346], [Task 356][t356], and [Task 404][t404]**.

## Rationale

After monetary and change viability are known,
reserve the complete automatic or caller-selected input set in one Store
transaction.

## Details

Generate one fresh cryptographically random 32-byte token inside
each admission, use the existing fixed-width lease owner representation, never
renew it, and reserve every selected input for one wallet-owned fixed 10-minute
PSBT funding duration. Define that duration as a package-private PSBT default
and do not add a caller-controlled duration in this task. The internal
admission helper must reject a non-positive duration before Store access even
though the public path passes the fixed default, and it must compute the exact
UTC expiration from the Store's injected clock. Return a wallet-owned funding
result containing packet, change index, typed token, expiration, and exact
selected set. Reconcile an ambiguous commit by reading the exact
set/token/expiration and return committed, absent, or typed indeterminate.
Exercise one Wallet and independent PostgreSQL Store transactions without
claiming multi-Manager/process support.

Exclusions: lease-set release, watched-change registration, caller-chosen
tokens, durable attempt manifests, advisory locks, cross-process fencing, and
general UtxoManager lease redesign.

## Suggested fix

Add one Store method that selects/reserves the final set in
one transaction using a generated fixed-width token, plus an exact set lookup
used to reconcile the operation before building the typed funding result.

## Verification

Barriers prove one token owns the entire set, contending
funders cannot share an outpoint, old tokens cannot validate later leases, and
exact/absent/partial/foreign/unavailable ambiguous states classify correctly.
The fixed public duration produces the exact Store-clock expiration, and a
focused internal-helper test proves a non-positive duration fails before Store
access. Invalid input amounts leave no lease.

## Definition of done

A successful PSBT funding result is published only
with one fresh token atomically owning its complete selected input set; an
ambiguous admission never guesses; the PR does not add cleanup behavior.

[t346]: 346-psbt-funding-metadata.md
[t356]: 356-defer-change-script-allocation.md
[t404]: 404-transaction-input-amounts.md
