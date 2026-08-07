# Task 327 — return a definitive passphrase-change result

**Status:** deferred, not scheduled. Raised by the post-delivery review of
#1272. The first Manager itest never changes a passphrase, and the pinned base
already permits cancellation while waiting for this request.

## Problem

`Wallet.ChangePassphrase` first sends an unbuffered request to the main loop,
then calls `waitForResp`. After the main loop accepts the request, the caller
can still receive `ctx.Err()` or `ErrWalletShuttingDown` instead of the
operation result. The Vault may meanwhile commit the new passphrase, leaving
the caller unable to tell which credential is authoritative.

The current method comment claims cancellation is possible only before
acceptance and promises a definitive result afterwards. That claim is false.
The three restored interruption tests preserve the opposite pinned-base
behavior.

## Why deferred

The delivered milestone routes lifecycle operations through the Vault and
removes `walletdb.Update/View`; it does not redesign Controller request
completion. The immediate #1272 correction should remove the false method
comment rather than silently widening the milestone.

Task 10002 is useful reference material for the broader request-completion
problem, but it is not an executable owner. This task deliberately covers only
the credential mutation whose ambiguous result can make a caller retain the
wrong passphrase.

## What to do

1. Keep context and wallet-shutdown cancellation while submitting the request.
2. Once `sendReq` succeeds, wait for the buffered response and return that
   definitive result. Inline the receive; do not add a one-line
   `waitForAcceptedResp` helper.
3. Define acceptance as the successful unbuffered-channel handoff to the main
   loop. Verify that every accepted request always publishes one buffered
   response, including Vault and lifetime-context failures.
4. Do not change `Unlock`, `Lock`, rescan, or general Controller completion in
   this task.

## Tests

- Cancellation before acceptance still returns the context error.
- Cancellation immediately after acceptance cannot hide a successful
  passphrase rotation.
- A Vault error after acceptance is returned even if the caller context or
  wallet lifetime is canceled concurrently.
- The test must prove which passphrase opens the Vault after the result; a
  channel-only assertion is insufficient for the success case.

Use focused unit tests. Add no itest scenario.

## Definition of done

1. An accepted passphrase mutation has exactly one caller-visible result.
2. Pre-acceptance cancellation remains supported.
3. No generic request helper or broader Controller lifecycle redesign is
   introduced.
4. The method documentation and tests state the same cancellation boundary.

## Evidence

Post-delivery #1272 review thread on `wallet/controller.go`:
<https://github.com/btcsuite/btcwallet/pull/1272#discussion_r3698807713>.
