# Task 280: Address PR 1267 Review Comments

Status: Archived. Completed.

Archive note: the missing-scope `ListAccounts` regression test is present on
`review-base-fixes`, and verification passed on a detached `review-base-fixes`
worktree:

- `go test ./wallet/internal/db/kvdb -run TestListAccountsMissingScopeReturnsEmpty`
- `go test ./wallet/internal/db/kvdb`

PR: https://github.com/btcsuite/btcwallet/pull/1267

## Process Note

When unresolved review comments need follow-up, create a task file under
`.lgtm/` that includes:

- the review comment reference;
- the assessment of whether the comment is valid/actionable;
- the proposed fix;
- the verification needed before resolving the thread.

## Review Comments

1. Copilot review thread:
   https://github.com/btcsuite/btcwallet/pull/1267#discussion_r3413733864

   Location: `wallet/internal/db/kvdb/accountstore_listaccounts.go:200-206`.

   Comment summary: `ListAccounts` now returns an empty result for an unknown
   scope, but there is no kvdb unit test covering the
   `ErrScopeNotFound -> empty list` path.

2. yyforyongyu review thread:
   https://github.com/btcsuite/btcwallet/pull/1267#discussion_r3418330093

   Location: `wallet/internal/db/kvdb/accountstore_listaccounts.go:201`.

   Comment summary: add a regression test asserting a missing-scope
   `ListAccounts` query returns an empty result with a nil error, preserving
   parity with SQL backends.

## Assessment

The comments are valid and actionable.

The implementation in `selectScopedManagers` appears correct: when
`FetchScopedKeyManager` returns `waddrmgr.ErrScopeNotFound`, the helper returns
`nil, nil`. The caller then walks no scoped managers, skips balance attachment
because no accounts were collected, and returns an empty slice with no error.

However, the exact behavior changed by PR #1267 is not covered by an existing
kvdb unit test. Existing kvdb tests cover filtering by a known scope and
missing-scope behavior for imported-account dry runs, but not `ListAccounts`
with an unregistered scope.

## Proposed Fix

Add a kvdb unit test in `wallet/internal/db/kvdb/accountstore_test.go`.

Suggested test shape:

```go
func TestListAccountsMissingScopeReturnsEmpty(t *testing.T) {
	t.Parallel()

	store, _, cleanup := newAccountStoreFixture(t)
	t.Cleanup(cleanup)

	scope := db.KeyScope{Purpose: 1234, Coin: 0}
	accounts, err := store.ListAccounts(t.Context(), db.ListAccountsQuery{
		Scope:       &scope,
		SkipBalance: true,
	})
	require.NoError(t, err)
	require.Empty(t, accounts)
}
```

Use `SkipBalance: true` so the test stays focused on scope filtering rather
than balance attachment.

## Verification

- `go test ./wallet/internal/db/kvdb -run TestListAccountsMissingScopeReturnsEmpty`
- `go test ./wallet/internal/db/kvdb`

## Acceptance Criteria

- The missing-scope `ListAccounts` behavior is covered by a kvdb unit test.
- The test fails on the old behavior and passes with PR #1267's implementation.
- Both unresolved review threads can be resolved with a comment pointing to the
  new regression test.
