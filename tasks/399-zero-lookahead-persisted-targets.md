# Task 399 — isolate persisted targets in zero-lookahead recovery

Component: **Syncer recovery initialization and account-scoped Store reads**.

Type: **Bug fix**.

Dependencies: **[Task 342][t342] and [Task 345][t345]**.

## Rationale

Zero lookahead must disable only new derivation, not persisted
watch synchronization. Current targeted scans load all wallet addresses and
watched outputs even after horizons are narrowed.

## Details

- For a full zero-lookahead scan, load every persisted `watch-chain` address,
  accountless raw import, watched outpoint, and required block identity while
  excluding key-only locators.
- For a targeted zero-lookahead scan, load only persisted watch-chain addresses
  and watched outpoints owned by an internal set of resolved durable account
  identities plus required block identities. Exclude unrelated accounts and
  all accountless imports without exposing those IDs publicly.
- Add only the narrow account-scoped address/outpoint Store queries needed by
  this initialization path.
- Preserve the same exact persisted target sets for positive lookahead; this
  task does not alter derivation counts or public target validation.
- Expose the narrow internal persisted-target loader consumed by Task 398 only
  after its full/targeted and zero/positive behavior is complete. Task 398
  solely owns the final public recovery request; this task adds no public
  selector or legacy adapter.

## Suggested fix

Split full and targeted persisted-target loaders around an internal durable
account-identity set and route both through mode-aware account-scoped queries
rather than the current wallet-wide address/outpoint calls. Task 398 later
resolves semantic selectors into that internal input.

## Verification

Zero and positive lookahead fixtures with multiple internal account identities,
numberless imported XPub, key-only locators, raw imports, and watched outpoints
prove the exact full and targeted sets on both SQL dialects. No zero scan
performs derivation, and no test needs the later public semantic selector.

## Definition of done

Zero lookahead preserves synchronization of exactly the
documented persisted targets and targeted recovery never widens to unrelated
wallet state. The internal loader is complete before Task 398 exposes semantic
target admission.

[t342]: 342-recovery-window-validation.md
[t345]: 345-durable-watch-registration.md
