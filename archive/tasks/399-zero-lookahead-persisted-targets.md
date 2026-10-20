# Task 399 — isolate persisted targets in zero-lookahead recovery

Component: **Syncer recovery initialization and account-scoped Store reads**.

Type: **Bug fix**.

Dependencies: **[Task 342][t342] and [Task 345][t345]**.

## Rationale

Zero lookahead should disable new derivation, not synchronization of already
persisted wallet targets. Targeted recovery currently reads wallet-wide
addresses and watched outputs, so narrowing the requested accounts does not
actually narrow the scan.

## Details

- A full zero-lookahead scan includes persisted addresses from ordinary
  accounts, accountless raw imports, watched outpoints, and required block
  identities. It excludes children of `NoChainSync` accounts.
- A targeted zero-lookahead scan includes only persisted addresses and watched
  outpoints owned by the resolved durable account set, plus required block
  identities. It excludes unrelated accounts, accountless imports, and
  `NoChainSync` children.
- Add only the wallet-owned account-scoped Store reads required for those
  result sets. Do not expose durable account IDs publicly.
- Preserve the same persisted-target selection under positive lookahead; this
  task does not change derivation counts or public target validation.
- Task 398 owns the public semantic recovery selector.

## Suggested fix

Filter the existing persisted-target loading by recovery mode and an internal
resolved account set. Query decomposition is flexible if the resulting target
sets and Store ownership remain the same.

## Verification

SQLite and PostgreSQL fixtures containing ordinary accounts, imported XPub
accounts, `NoChainSync` children, raw imports, and watched outpoints prove the
exact full and targeted sets for zero and positive lookahead. Zero-lookahead
cases perform no derivation.

## Definition of done

Zero lookahead preserves every in-scope persisted target without deriving new
ones, and targeted recovery cannot widen to unrelated wallet state.

[t342]: 342-recovery-window-validation.md
[t345]: 345-durable-watch-registration.md
