# Task 286: Resolve Targeted Rescans By Account Name First

Status: Archived. Completed.

Archive note: `syncer.resolveScanTarget` now prefers non-empty
`AccountScope.Name`, returns not-found for unknown names without falling back to
account number, and retains the non-masked-number manager fallback when no name
is supplied. Verification passed:

- `go test ./wallet -run 'TestManagerCreateRejectsLoadedWallet|TestManagerCreateRejectsStartedWallet|TestManagerCreateReleasesGuardOnFailure|TestResolveScanTargets|TestStoreScanHorizonsTargeted|TestGetUtxo|TestFilterEligibleOutputsExcludesNonSpendable|TestImportPrivateKeyKVDBSucceeds'`

Source review: `.lgtm/reviews/gpt-review-tasks.md`, finding 3.

## Finding

Targeted rescan resolution still performs an account-number lookup before using
`waddrmgr.AccountScope.Name`. `AccountScope.Name` is documented as the stable
disambiguator for imported accounts that share account number `0` with the
default derived account.

For a target such as `AccountScope{Account: 0, Name: "imported-xpub"}`, a
number-first lookup can resolve the default derived account before the imported
account name is considered.

## Rationale

The public rescan target already has the data needed to avoid the collision.
When `Name` is set, it is more specific than the masked account number and must
be treated as the source of truth.

The current fallback for imported-xpub accounts addressed by their internal
non-masked number is still useful, but it does not cover the public API case
where a caller supplies the durable name with account number `0`.

## Detailed Fix Plan

1. Update `syncer.resolveScanTarget` to prefer `target.Name` when it is not
   empty.

2. For a named target, call `Store.GetAccount` with `Name` and `Scope`, not with
   `AccountNumber`. Return a `scanTarget` carrying the resolved `AccountName`.

3. Only use account-number lookup when `target.Name == ""`.

4. Preserve the existing fallback to the identity-aware legacy manager for
   imported-xpub targets addressed by non-masked internal account number.

5. Preserve the special handling for the keyless imported-address bucket. If the
   target explicitly names that bucket, make the behavior intentional and tested
   rather than accidentally resolving by account number.

6. Audit comments around `AccountScope`, `resolveScanTarget`, and
   `storeScanHorizonAccount` so they consistently say name is preferred whenever
   available.

## Tests

- Add a unit test with both the default derived account and an imported-xpub
  account. Call `resolveScanTargets` with
  `AccountScope{Scope: scope, Account: 0, Name: "imported-xpub"}` and assert the
  resolved target names the imported account, not the default account.
- Add a negative test for an unknown non-empty `Name`: it should return a clear
  account-not-found error rather than falling back to account `0` and scanning
  the default account.
- Keep the existing test for imported-xpub targets addressed by their non-masked
  internal account number.

## Verification

- `go test ./wallet -run 'TestStoreScanHorizonsTargeted|TestResolveScanTargets'`
- `go test ./wallet`

## Acceptance Criteria

- `AccountScope.Name` is always preferred over `AccountScope.Account` during
  targeted rescan target resolution.
- A named imported-xpub target with account number `0` cannot resolve to the
  default derived account.
- Existing non-masked-number imported-xpub target behavior remains covered.
