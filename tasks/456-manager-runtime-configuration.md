# Task 456 — centralize managed Wallet runtime configuration

Component: **ManagerConfig and managed Wallet construction**.

Type: **Refactor**.

Dependencies: **[Task 334][t334]**.

## Rationale

An aggregate Manager start must recreate every durable Wallet without a caller
supplying a separate `wallet.Config` for each one. The maintained Manager API
still accepts Wallet-scoped chain and runtime policy even though the Manager
owns the shared chain source, network, Store, and lifecycle.

## Details

- Move the maintained runtime policy needed after restart into one immutable
  `ManagerConfig`: chain-source construction, network, synchronization method
  and retry policy, recovery window, auto-lock policy, and filter threshold.
- Move the legacy kvdb runtime alias and public-passphrase input needed to open
  its sole Wallet into the documented kvdb-only Manager configuration boundary.
  Require kvdb Create identity to match that configured alias; do not persist
  restart-stable alias metadata in the legacy database.
- Deep-copy network parameters, byte slices, and other mutable value input at
  Manager construction. Retain one source factory as the Manager's exclusive
  construction boundary, and expose no retained configuration pointer or byte
  slice through Wallet information APIs.
- Reduce maintained Manager Create input to durable Wallet identity and
  initialization material. A created Wallet cannot select another chain source,
  network, backend, or runtime policy.
- Preserve `wallet.Config` only for deprecated non-Manager constructors that
  remain outside the maintained SQL design. Do not add an adapter between the
  old and final Manager APIs solely for compatibility.
- Keep source-instance ownership out of this task. Before Task 423, existing
  internal Wallet setup may consume the Manager-owned factory through its
  maintained adapter; Task 423 becomes the sole caller that constructs the one
  final SQL Manager source. Do not expose that transition in public config.
- Update maintained Wallet construction, callers, mocks, and tests for the
  final ownership shape without starting workers or changing lifecycle.
- Exclude durable Wallet enumeration, Manager Start/Stop, Create activation,
  config persistence, runtime manifests, CLI, RPC, lnd, and daemon changes.

## Suggested fix

Extend `ManagerConfig` with the final Manager-owned runtime fields and retain
one validated immutable snapshot. Build managed Wallet configuration internally
from that snapshot plus durable Wallet identity. Keep deprecated constructors
on their existing isolated configuration path.

## Verification

- Mutate every copied caller-owned pointer, byte slice, and nested runtime
  option after `NewManager` and prove later managed Wallet construction is
  unchanged. Prove no Wallet can replace or expose the retained source factory.
- Build two SQL Wallets and the one modern-kvdb Wallet from the same Manager
  snapshot and prove their shared runtime policy is identical.
- Reopen modern kvdb with its configured runtime alias and prove no alias row or
  other persistent metadata was introduced.
- Reject missing or conflicting Manager runtime input before Store mutation.
- Compile maintained callers, interfaces, and mocks without a per-Wallet chain
  or runtime-policy argument to Manager Create.
- Confirm deprecated constructors still build without importing their config
  path into managed Wallet construction.
- Confirm Task 423 can change internal source construction from the pre-cutover
  adapter to one Manager source without another public configuration change.

## Definition of done

- Manager owns one validated immutable runtime policy sufficient to recreate
  every durable Wallet after restart.
- Maintained Create input contains only durable Wallet identity and
  initialization material.
- Managed Wallets cannot override Manager-owned chain, network, backend, or
  runtime policy.
- The PR changes no lifecycle, persistence, runtime manifest, CLI, RPC, lnd,
  or external-daemon behavior.

[t334]: 334-shared-chain-state.md
