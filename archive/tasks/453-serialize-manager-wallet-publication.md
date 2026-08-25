# Task 453 — serialize Manager Wallet assembly

Component: **Manager Wallet assembly and cache installation**.

Type: **Bug fix**.

Dependencies: **None**.

## Rationale

Concurrent cold Loads for one wallet can both miss the Manager cache, assemble
separate Wallets, and race to install different pointers. Create can likewise
commit a durable wallet and race Load before installing its runtime pointer.
Callers can then receive different live Wallet identities for the same durable
wallet.

Create and Load are infrequent Manager control-plane operations. They do not
need parallel per-name admission. Serializing them across one Manager gives the
cache, durable operation, and runtime pointer one simple ownership boundary.
This assembly race is independent of starting or stopping a Wallet and must be
closed before Task 368 adds Manager-owned lifecycle state.

## Details

- Serialize every Manager Create and cold Load with one Manager-owned lock.
  Different wallet names on the same Manager intentionally assemble
  sequentially.
- Preserve validation and caller-only key derivation outside the lock. Hold the
  lock from the durable Create operation, or the Load cache decision, through
  backend assembly and installation of the exact runtime Wallet pointer.
- Keep the cache as `map[string]*Wallet`. Do not publish nil or in-flight cache
  entries and do not add per-name channels, result latches, operation records,
  or helper goroutines.
- A Load cache hit returns the exact installed pointer. Concurrent cold Loads
  serialize; the first successful call installs one pointer and later callers
  return it without assembling another Wallet.
- Create and Load share the same serialization boundary. Load cannot observe a
  newly committed durable wallet before the Create caller installs its runtime
  pointer.
- Release the lock on every failure. Do not cache errors; a later Create or Load
  retries the durable operation normally.
- Preserve backend parity. SQL and the legacy one-wallet kvdb backend use the
  same Manager admission rule; cross-name kvdb admission remains Task 331's
  scope.
- Exclude lifecycle state, terminal replacement, Manager Close admission,
  Create and Load context cutoffs, durable duplicate-create classification,
  generic operation registries, persistent manifests, and cross-process
  fencing.

## Suggested fix

Use the Manager's cache mutex as the assembly admission lock. After validation,
Create acquires it before `backend.create` and keeps it until the new Wallet is
installed. Load acquires it before checking the cache; on a miss, it keeps the
lock through `backend.load`, Wallet assembly, and cache installation. String and
other cache readers use the matching read lock.

Keep the implementation synchronous. There is no losing assembly, joined
result, or transient cache state to explain. Task 368 may replace the bare
pointer with a lifecycle runtime entry after assembly has completed, but Task
453 does not introduce that entry early.

## Verification

- Barrier concurrent same-name cold Loads on kvdb, SQLite, and PostgreSQL.
  They return one exact pointer and perform one backend assembly.
- Hold a Create after durable commit and prove a racing Load cannot enter its
  cache decision or backend Load before Create installs the runtime pointer.
- Force a cold assembly to fail, then prove the lock is released and a later
  Load can retry and install a Wallet.
- Prove different-name cold Loads on one Manager are admitted sequentially.
- In the backend integration case, use one setup Manager to create durable
  state and a second Manager with an empty runtime cache to exercise a true
  concurrent cold Load.
- Run the focused Manager assembly races under
  `make unit-race pkg=wallet case=TestManagerWalletAssembly`, followed by
  `make unit pkg=wallet`.

## Definition of done

- Manager Create and Load have one synchronous assembly owner at a time.
- Concurrent same-name cold Loads return one exact Wallet pointer, and Create
  cannot be overtaken between durable commit and cache installation.
- Failed operations leave no cache sentinel or stranded waiter and a later
  caller can retry.
- The cache contains only fully assembled Wallet pointers.
- The PR contains no lifecycle, Close, cancellation, durable duplicate-create,
  manifest, or cross-process behavior.
