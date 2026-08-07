# Task 342 — accept bounded recovery windows

Component: **wallet configuration**.

Dependencies: **None**.

## Rationale

Configuration rejects every recovery window below 20 even though recovery
already operates with smaller windows. It also has no public upper bound, so
callers cannot validate expensive requests before Store or chain work.

## Details

- Export `wallet.MaxRecoveryWindow = 100_000`.
- Accept every `RecoveryWindow` from zero through that maximum unchanged.
- Reject maximum-plus-one with the existing wallet-owned invalid-parameter
  contract during configuration validation, before Store or chain access.
- Remove the artificial minimum from the maintained Manager configuration.
- Do not change scan derivation, target selection, aggregate budgeting,
  invalid-child behavior, or signed RPC validation. [Task 363][t363] owns
  aggregate derivation work; [Task 397][t397], [Task 398][t398], and
  [Task 399][t399] own the semantic target contract, admission, and
  persisted-target loading.

## Suggested fix

Replace the minimum check in `Config.validate` with one checked public maximum
and update callers and fixtures that use `MinRecoveryWindow` merely to satisfy
validation. Keep RPC negative-value handling outside the unsigned Wallet
configuration.

## Verification

- Focused configuration tests accept 0, 1, 2, 19, 20, and
  `MaxRecoveryWindow` unchanged.
- Maximum-plus-one returns the wallet-owned invalid-parameter error without
  Store or chain access.
- Existing small-window recovery fixtures remain valid, and RPC retains its
  separate negative-value check.

## Definition of done

- Unified Wallet configuration accepts `0..MaxRecoveryWindow` and rejects
  the first excess value before dependency access.
- No scan-work, target, derivation, or RPC policy changes in this PR.

[t363]: 363-recovery-scan-work-bound.md
[t397]: 397-semantic-recovery-targets-adr.md
[t398]: 398-semantic-account-recovery-targets.md
[t399]: 399-zero-lookahead-persisted-targets.md
