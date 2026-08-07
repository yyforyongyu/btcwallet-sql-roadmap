# Task 1000: Complete btcwallet SQL role readiness

Status: Proposed, **deferred** (renumbered from 300 to the 1xxx deferred tier on
2026-07-29). Reference only — not worked on in the current round.

Target branch: `wallet-default-sqlite`.

Evidence tip: `24e1343c9a3e5b6271efef200b1e277b1459dc0b`.

Evidence frozen: 2026-07-28.

The branch was rewritten from `2684a5712` to `d27179098`, then to
`967eeb3b7`, `493c24f7e`, `69128d9f3`, and then to this evidence tip. S0 must
range-diff `24e1343c9` against `1f88969ae`, `2684a5712`, `d27179098`,
`967eeb3b7`, `493c24f7e`, and `69128d9f3` and re-establish every tip-anchored
claim.

If the branch is force-pushed after this freeze, implementation stops until S0
records the successor SHA, range-diffs every reviewed tip, and re-pins here.

## Purpose

Finish the btcwallet-owned work required for a fresh lnd node to use the
role-based SQLite or PostgreSQL wallet without a walletdb sidecar or legacy
`wallet.Interface` dependency.

This task is intentionally about the wallet contract and wallet correctness.
It does not implement the lnd adapter. The lnd integration is allowed to depend
on this task only after each required wallet behavior has backend conformance
evidence.

The work is not a rewrite of the SQL wallet. The Store, Manager, Controller,
syncer, transaction creator, signer, PSBT manager, SQLite backend, and
PostgreSQL backend already exist. The remaining work is to close public-contract
holes, correct behavior exposed by comparative testing, define safe failure
semantics, and prove the result against real chain backends.

## Decision Context

The route comparison selected the role-based wallet because it already keeps
chain RPC, signing, and policy work outside Store transactions and performs
materially better in fair full-block SQLite benchmarks. The compatibility-first
route failed its transaction-boundary stop condition because completing it
would require prepare/commit/reconcile semantics throughout the legacy wallet.

Selecting the role route did not make the current branch production-ready. The
comparison exposed the following wallet defects or missing contracts:

- SQL migration versions do not identify the schema family, and the database
  does not durably bind itself to one Bitcoin genesis hash.
- Public account and status APIs still expose internal DB types.
- Recovery-window validation rejects values lnd already uses.
- Wallet script persistence still exposes legacy crypto-key-type selection.
  The real-vault SQLite watch-only round trip succeeds, but Manager restart and
  PostgreSQL conformance remain unproven, and SQL/KV locked reads differ.
- Watch-only Controller behavior differs between KV and SQL.
- A connected-block tip may become visible before its relevant transaction and
  balance effects.
- Custom scopes and exact sparse accounts cannot be created through the public
  account role.
- Forced address allocation is count-one, cannot safely skip invalid BIP32
  children, and does not support exact key-only accounts.
- Returning an existing unused address does not repair an earlier chain-watch
  registration failure.
- PSBT funding reconstructs packet maps and loses caller metadata.
- PSBT signing rejects persisted scopes based on the network's conventional
  coin type rather than proving wallet ownership from persisted identity.
- Funding-created change may be returned before its script is registered with
  the chain backend.
- SQL branch invalidation exists internally but is not exposed as a narrow,
  idempotent public operation.
- Store commit ambiguity is classified, but wallet operations do not reconcile
  the durable outcome before publishing results or side effects.

## Revalidated Scope

Several older planning claims are stale at the evidence tip and must not create
work in this task:

- Manager now has durable create manifests and completion state, serialized
  per-name create/load/teardown, delayed cache publication, single-flight load,
  and stopped-wallet eviction. Preserve that architecture; complete only its
  deterministic prefix classification, guarded normalization, visibility,
  resume, and safe-abort/cutover rules required by the SQL lifecycle.
- lnd will construct partial chain control before using Manager. Do not add
  deferred chain attachment to btcwallet.
- The modern `creating`/`ready` marker is lnd-owned. Do not add an lnd readiness
  marker to btcwallet.
- lnd must define a private composition of modern wallet roles. Do not embed the
  roles into the legacy `wallet.Interface`.
- UTXO filtering, multi-scope funding policy, lease-result enrichment, account
  import previews, lnd-compatible ECDH, and transaction-subscription polling
  are lnd-side compositions.
- Existing-node KV-to-SQL migration is a later milestone.
- Legacy daemon and JSON-RPC conversion is not required for the fresh-node lnd
  milestone.

### Milestone-Critical Versus Forward Hardening

The fresh-lnd milestone assumes one btcwallet process per wallet database.
Milestone-critical work includes durable identity, public roles/recovery,
context-bounded lifecycle, migration safety, single-writer atomic chain state,
account/allocation/watch readiness, PSBT/funding/signing, publication,
invalidation, reconciliation, and SQLite/PostgreSQL chain conformance.

Simultaneous processes sharing one role schema are forward hardening:
participant unanimity/common-prefix arbitration, takeover, owner replacement,
and multi-owner fencing. This task still treats them as release requirements.
Deferral requires an explicit follow-up plus fail-closed single-active-process
enforcement and corresponding ledger/PR/CI/DoD rewrites; they cannot be silently
omitted while claiming shared-PostgreSQL or multi-process support.

## Architectural Constraints

Every change in this task must preserve the accepted architecture:

1. ADR 0002 keeps lifecycle in Controller, chain work in syncer, and state in
   the orthogonal wallet state model.
2. ADR 0004 keeps targeted rescans separate from the global synced watermark.
3. ADR 0005 keeps historical discovery explicit: imports never start a rescan.
   This task updates the ADR so a public live-watched import may persist first,
   then require bounded idempotent registration before returning a usable
   result. Store-only/offline import remains database-only.
4. ADR 0006 keeps immutable transaction history, explicit status transitions,
   atomic invalidation, and leases as the durable selection primitive.
5. ADR 0010 keeps script and key cryptography behind the wallet-scoped vault.
   Store rows remain opaque encrypted bytes.
6. ADR 0011 keeps SQL address used-ness derived from retained transaction
   history. Do not add an `addresses.used` column.
7. ADR 0012 keeps each SQL wallet uniformly spendable or uniformly watch-only.
   Do not introduce a watch-only account into a spendable SQL wallet.
8. ADR 0013 keeps account, address, and derivation identity normalized. The
   exact external-XPub design below requires an explicit ADR update because the
   current ADR says imported XPub accounts never have account numbers.
9. Chain calls, signing, callbacks, and notifications must not occur inside a
   Store transaction.
10. Cache publication and external side effects happen only after a definite or
    reconciled commit.
11. SQL source files are edited first and generated with `make sql`; generated
    sqlc files are never edited manually.
12. One SQL database is bound durably to the role-wallet schema family and one
    Bitcoin genesis hash before any wallet row is read or mutated.
13. Migrations become immutable when the first independently usable PR can
    record them as applied. Every later schema change uses a new forward
    migration and upgrade fixture.
14. No vault, waddrmgr crypto, derivation, or cache callback runs inside a Store
    write transaction.

## Issue Ledger

| ID | Severity | Issue |
|---|---|---|
| BWR-000 | Critical | SQL schema family and network identity are not durable |
| BWR-001 | High | Public role types and errors leak internal DB details |
| BWR-002 | High | Recovery-window validation rejects valid small values |
| BWR-003 | Conformance | Legacy cipher coupling and incomplete SQL evidence |
| BWR-004 | High | Watch-only lifecycle semantics differ by backend |
| BWR-005 | High | Synced tip can precede transaction and balance state |
| BWR-006 | Critical | Exact custom and path-bound XPub accounts are absent |
| BWR-007 | Critical | Bulk allocator lacks batch and invalid-child safety |
| BWR-008 | High | Key-only intent and chain registration are not durable |
| BWR-009 | High | Funding loses or misassociates PSBT metadata |
| BWR-010 | High | PSBT signing applies a network-global coin-type rule |
| BWR-011 | High | Funding can return unwatched change |
| BWR-012 | High | Branch-scoped transaction invalidation is not public |
| BWR-013 | Critical | Ambiguous commits lack operation-level reconciliation |
| BWR-014 | Gate | Complete SQLite and PostgreSQL chain evidence is absent |
The issue ledger belongs in this task while implementation is in progress. The
lnd gap register must be refreshed when a wallet issue is closed, but lnd-only
entries remain outside this task.

`Gate` means required release evidence rather than a known production defect.

`Conformance` means required contract, architecture, and backend evidence rather
than a reproduced production defect.

## Common Correctness Rules

All work packages use the following rules.

### Public Boundary

- Public role signatures contain only wallet-owned or dependency-owned public
  types.
- Public control-flow errors are stable wallet sentinels or typed errors.
- Internal DB and runtime errors may appear in log/debug text, but external
  callers must not need to import an internal package for `errors.Is` checks.
- Add only generic wallet operations. Do not add a method whose sole purpose is
  to preserve an lnd implementation detail that can be composed from existing
  roles.

### Store Boundary

- Validation and pure preparation happen before a write when possible.
- A Store method owns one complete durable invariant.
- Backend adapters use the shared ops pattern for semantic parity.
- SQLite and PostgreSQL implement the same observable result and error
  behavior.
- KV receives regression coverage whenever a shared wallet role or Store
  contract changes.

### Prepared Derivation Mutations

- Read an immutable counter/state snapshot in a read transaction.
- Derive/neuter keys, construct scripts, and encrypt candidate material outside
  write admission and Store transactions.
- Commit with a CAS over the expected counter plus relevant scope, account,
  schema, discovery, runtime, and chain-view state.
- A pre-COMMIT CAS miss writes no row, intent, outbox event, manifest state, or
  cache entry. Zero/discard prepared secrets, then reread and reprepare.
- Once COMMIT begins, retain the exact prepared candidate until ambiguity
  reconciliation proves rollback; never re-encrypt during reconciliation.
- Publish waddrmgr/runtime caches only after definite or reconciled commit.

### Modern Chain Side Effects

PR 2 lands before the SQL runtime foundation. Preserve legacy
`chain.Interface`, but add context-aware modern registration/publication:

- Registration carries exact scripts/outpoints, stable wallet `SubscriptionID`,
  immutable persisted `DeliveryID`, payload digest/mode/watch generation,
  diagnostic origin epoch, and current dispatch epoch used only for fencing.
  Backend dedup keys on subscription/delivery, so successor replay uses the same
  delivery with its current epoch; changed content conflicts and delayed old
  epochs fail stale.
- Complete/additive registration is one logical transaction. Validate
  aggregate counts/bytes/chunk order/digest, stage every bounded chunk while
  preserving the old active set, establish the candidate mempool/live barrier,
  then atomically activate. Pre-activation failure discards staging. Every stale
  generation, including additive, no-ops/rejects; equal generation/delivery/
  digest replays idempotently. Unacked post-activation cancellation is unknown.
  Backends unable to stage/activate/barrier return unsupported without partial
  exposure.
- Successful registration includes a backend-observable mempool barrier: return
  or queue existing matching transactions available to that backend. A backend
  unable to establish it returns typed unsupported; this is not a claim of
  network-global mempool completeness.
- Publication outcomes are `accepted`, `already_known`, `definitive_reject`,
  `retryable`, `unknown`, or `not_attempted`. Registration outcomes are
  `registered`, `unsupported`, `stale`, `conflict`, `retryable`, `unknown`, or
  `not_attempted`. Unsupported is terminal capability failure. Stale proves no
  backend mutation and fences that dispatch; the event remains pending unless a
  newer activated snapshot proves it subsumed. Conflict means the same identity
  carried different content, quarantines the registration invariant, and never
  advances its cursor. Original causes remain available for diagnostics/
  `errors.Is`, but only the enum drives durable state and only definitive
  publication rejection permits invalidation.
- Freeze immutable `BackendCapabilities` per attached backend generation,
  established during backend setup and readable without a backend operation.
  It advertises arbitrary-tx status proof. Ownership-free enqueue consults this
  local generation-bound capability before attempt creation; a false value
  returns unsupported with no durable row.
- Add a context-aware publication-status proof for arbitrary txid/full bytes.
  Typed outcomes are canonical `confirmed(height, block hash, membership
  proof)`, `unconfirmed_backend_known`, `not_found`, `reorged`, `unsupported`,
  `retryable`, or `unknown`. Proofs bind backend generation and canonical-view
  epoch. btcd/bitcoind map authoritative RPC results; neutrino returns
  unsupported when an ownership-free/key-only transaction is outside its watch
  filter and cannot be proven. If a backend advertised support but an actual
  post-attempt proof returns unsupported, retain the attempt, fence that backend
  generation as capability-inconsistent, and fail closed; never erase the row
  or fabricate confirmation.
- Policy-mutable, missing-parent, mempool-conflict/full, fee, non-final,
  unclassified/future, transport, cancellation, and uncertain outcomes retain
  publication state. Cancellation before dispatch is not-attempted; unacked
  dispatch is unknown. Registration failure never invalidates.
- Registration unsupported writes terminal-unsupported disposition, advances
  contiguous cursor/completes intent, and leaves readiness false. It returns a
  stable capability error without invalidation/quarantine. During activation it
  aborts/fences starting ownership; a later capable backend may establish state.
- Every call has caller cancellation plus a hard bound and leaves no detached
  goroutine/request. btcd, bitcoind, and neutrino provide exhaustive mappings;
  dependency support lands/tags before PR 2 where needed.

PR 2 tests blocked calls, before/after-dispatch cancellation, every known RPC
class plus unknown defaults, exact scripts/outpoints, chunk failure and
cancellation, old-set preservation, successor replay, stale additive,
unsupported disposition, pre-existing mempool matches, bounds, replay, and
backend compile assertions. Status tests cover confirmed/unconfirmed/not-found/
reorg/unsupported and generation changes on all three backend adapters. A
capability-false preflight writes no attempt; advertised-true then unsupported
retains the attempt and fences the inconsistent generation.

### Backend Capability Matrix

S0 records this matrix against the frozen interfaces and each implementation PR
updates it:

| Capability | SQLite/PostgreSQL | Modern KV adapter |
|---|---|---|
| Public account/status values | Required | Required adapter |
| Small recovery windows | Required | Required shared behavior |
| Sequential allocation | Required | Numbering, invalid-child, exhaustion |
| Exact path-bound XPub accounts | Required | Public unsupported |
| Durable key-only discovery mode | Required | Public unsupported |
| Bulk allocation | Required | Count, bounds, order, invalid-child, exhaustion |
| Fingerprint absent versus present-zero | Required | Required side metadata |
| SQL mutation intents | Required | Not applicable |
| Public branch invalidation | Required | Public unsupported |
| PSBT/funding shared behavior | Required | Regression parity |
| Ambiguous COMMIT reconciliation | Required | Not applicable |

The KV backend is not required to adopt SQL persistence or lnd's modern wallet
path. Its existing account, address, transaction, and lock behavior remains the
regression oracle except for the explicit shared rows above.

### Ambiguous Commit Boundary

An ambiguous commit means COMMIT returned an error while the database may have
committed the complete transaction. It is not an ordinary retryable error.

Every mutating operation added or changed by this task must define:

1. A random operation ID and SQL mutation intent written atomically with the
   candidate mutation.
2. The expected old-state digest and candidate durable result captured before
   COMMIT.
3. The result rows or operation payload linked to that operation ID.
4. The exact rows and fields used to recognize the candidate after COMMIT.
5. The old-state facts that prove the transaction rolled back when no intent
   exists.
6. The mismatch that means another writer won or durable state is unhealthy.
7. Which cache or external side effect may publish before the intent is marked
   complete.
8. The owner-capability transition, if any. Claim, renew, release, fence, and
   handoff for `create-owner`, `starting-owner`, `active-owner`,
   `cutover-owner`, and `maintenance-owner` each has a closed transition ID,
   expected prior owner/epoch/state, candidate owner/epoch/state, and fresh-read
   reconciliation. No role or side effect publishes until an ambiguous claim is
   proven; an ambiguous release remains conservatively owned/fenced until
   proven. Natural-ID control-plane CAS is allowed only by the explicit journal
   exemption and must still define the same four outcomes.

There is no generic blind retry. Reconciliation uses a fresh transaction and
returns one of four outcomes:

- **Committed:** every candidate fact matches; return success and publish.
- **Rolled back:** the old state still matches; retry only if the operation's
  policy explicitly permits it.
- **Conflicted:** a valid competing state exists; return the public conflict.
- **Unknown:** neither candidate nor old state can be proven; return the public
  ambiguous error and suppress all side effects.

An absent intent proves rollback only while the originating Store session is
still active and unfenced and its operation sequence is above that session's
durable prune watermark. A stale or fenced caller never turns absence into
rollback proof.

Partial presence after an operation that is supposed to be atomic is an
integrity failure, not a successful partial result.

## S0: Freeze The Current Contract And Evidence

### Rationale

The gap register was audited against `1f88969ae`, and earlier task drafts were
verified at `2684a5712`, `d27179098`, `967eeb3b7`, and `493c24f7e`. The branch
was rewritten again to `69128d9f3` and then `24e1343c9`. Comparative tests also
live in an experimental worktree. A production plan must distinguish current
defects from stale assumptions.

### Work

1. Confirm `24e1343c9a3e5b6271efef200b1e277b1459dc0b` is still the
   target-branch HEAD. Range-diff it against
   `69128d9f3dd17097adb302ff1bc2a3591b12320e`,
   `493c24f7eba4186e605df78b7eae63620abbb39f`,
   `967eeb3b7f3c34a2c6561ef967bd8da19aa48b53`,
   `d271790988c7c5b2d13d6caf4e063c6431d175ce`,
   `2684a5712731d30a831025fa2d48549d74c9eac3`, and
   `1f88969aed9d44c8a8af06aa343789b90fe11983`. If HEAD moved, re-pin first.
2. Record every rewrite classification since the last reviewed tip. For
   `493c24f7e -> 69128d9f3`, all 26 commits map one-for-one; 22 are
   patch-identical and four changed: account-secret adapters (`605d73cf9`),
   signer routing (`6b9542052`), runtime Store sharing/security (`7d6a32549`),
   and reset coordination (`d3af81249`). The direct 18-file `+1662/-379` delta
   adds account-store commonization/tests, runtime-registry identity/conflict
   handling, reset operation sharing/tests, and rollback/runtime tests. Recheck
   every overlapping claim; no migration/toolchain file changed.
   For `69128d9f3 -> 24e1343c9`, all 22 commits map one-for-one; 19 are
   patch-identical and three changed: signer routing (`b9c990520`), runtime
   Store sharing/security (`668cc97a8`), and reset coordination (`c04dadb89`).
   The direct six-file `+708/-71` delta strengthens runtime-registry security
   policy comparison/tests, reset ordering/liveness/error fanout, and signer
   secret handling/comments. Revalidate those planned surfaces; migrations and
   toolchain files remain unchanged.
3. Classify toolchain work: all seven `go.mod` files, `.golangci.yml`, and
   `tools/Dockerfile` already use Go 1.25.11. `AGENTS.md` and the main workflow
   still name 1.24.6; repository-wide `GOTOOLCHAIN=local` remains PR 1 work.
4. Revalidate the rewritten runtime Store registry, FIFO SQL `WriteGate`, reset
   coordinator, create-retry authentication, and existing forward-only
   sync-tip guard before changing overlapping code. BWR-005 extends the current
   greater-height guard with expected-tip and ancestry checks; it does not
   introduce forward-only protection from scratch.
5. Port the role behavior vectors from
   `.worktrees/impl-298-sqlite/itest/comparative_validation_test.go` into a
   maintained test location without importing salvage implementation code.
6. Confirm the current unlocked SQLite production-vault tapscript round trip,
   persisted non-secret identity, and locked internal
   `keyvault.ErrVaultLocked` result. Reproduce the missing public
   `ErrWalletLocked` translation plus
   the remaining Manager restart, PostgreSQL, watch-only lifecycle,
   live-confirmation, PSBT metadata, and persisted-scope signing gaps.
7. Change the live-confirmation test to wait on transaction state and balance,
   not merely wallet height. Preserve a separate assertion for the stronger
   synced-tip invariant described in BWR-005.
8. Inventory every role signature and control-flow error visible to an external
   package.
9. Inventory every Store mutation reachable from Manager creation, Controller
   lifecycle, accounts, addresses, imports, funding, transaction publication,
   synchronization, rollback, and startup maintenance. Record its ambiguity
   policy and external side effects before changing implementation.
10. Inspect the actual SQL and DB adapter filenames at the frozen tip. Do not
   carry provisional filenames from older snapshots into implementation PRs.
11. Update `wallet-default-sqlite-lnd-gaps.md` to remove stale Manager
    partial-create/deferred-chain assumptions, classify BWR-003 as conformance
    and architecture work, and add BWR-004/BWR-005 explicitly.
12. Keep `yyforyongyu/lnd#73` and the role lnd worktree as call-site evidence
    only. Do not design wallet APIs from compile failures alone.
13. Record the immutable production evidence tip before benchmark-only changes
    as `S0_SYNC_BASE_SHA`. Port
    `wallet/fair_sqlite_sync_benchmark_test.go` and a sync-capable
    `scripts/run-sql-role-benchmarks.sh` in a separate no-production
    `SYNC_HARNESS_SHA` commit that applies cleanly to base and candidate.
    It owns every literal fair-sync name and verifies production hashes are
    unchanged by harness application.

### Exit Gate

- Every issue in this task has a failing test, compile assertion, or exact code
  inspection at the selected tip.
- Stale issues are removed rather than carried as speculative work.
- The fair SQLite benchmark and same-host baseline are maintained artifacts,
  not files that exist only in an experimental worktree or temporary directory.
- No production behavior changes land in the evidence-only commit.

## BW0: Durable Database Identity

### BWR-000: Schema Family, Migration Namespace, And Genesis Binding

#### Issue And Rationale

At the frozen tip, the role schema is at migration version 11 and the latest
salvage schema inspected here is at version 14. Their numeric namespaces overlap
through version 11. The migration framework records only version and dirty state
in its default table, so a foreign family can appear current or ahead rather
than being rejected.

The runtime Store registry rejects a network mismatch only while one process
keeps the database open. After restart, the same SQL database can be opened with
different chain parameters because no durable genesis identity exists.

PostgreSQL adds a namespace concern. lnd and btcwallet may share one PostgreSQL
database, while current wallet tables and the default migration table are
unqualified. btcwallet must not collide with another application's tables or
migration state.

#### Detailed Fix

1. Give btcwallet a dedicated migration table name. Do not use the default
   `schema_migrations` table.
2. Store a durable metadata row containing:
   - a stable role-wallet schema-family identifier;
   - the supported schema generation;
   - the complete Bitcoin genesis hash;
   - the SQL backend family where useful for diagnostics.
3. For PostgreSQL, place all btcwallet tables, metadata, and migration state in
   a dedicated btcwallet schema. Set and verify the query namespace explicitly;
   do not rely on a caller's ambient `search_path`.
4. Run a side-effect-free preflight before WAL, mutating pragmas, migration
   driver construction, migration tables, or wallet mutation:
   - an empty namespace may be initialized;
   - a matching family/generation/genesis may open;
   - a dirty, newer, foreign, or wrong-network database fails closed;
   - a non-empty unmarked namespace is never assumed to be a role wallet.
    Inspect existing SQLite files read-only and PostgreSQL catalogs first, then
    repeat classification under the database lock before the first write.
   For SQLite, resolve the existing parent directory, reject a symlink database
   final component and files with multiple hard links, and use that canonical
   parent/basename for the runtime registry and control-lock namespace.
5. Coordinate bootstrap and migration with a database-level lock, not the
   process-local runtime registry or migrate driver lock:
   - SQLite uses one pinned connection and `BEGIN IMMEDIATE` for locked recheck,
     marker creation, migration, and final state transitions;
   - PostgreSQL holds a session advisory lock on the same dedicated pinned
     connection used for preflight and migration;
    - derive the PostgreSQL signed 64-bit schema key from the first eight bytes
      of SHA-256 over an unambiguously length-prefixed protocol version,
      server-returned `current_database()`, btcwallet schema, and lock kind
      `schema`; interpret it as big-endian two's-complement and never include
      wallet, Store session, create, process, DSN, host, port, user, or pooler
      identity;
    - bootstrap/migration exclusive locking, Store-lifetime shared leasing, and
      transaction-scoped shared schema fencing use that identical key with only
      the advisory lock mode differing;
   - session-level advisory locking requires session affinity and is unsupported
     through transaction-mode PgBouncer unless one encompassing transaction can
     use `pg_advisory_xact_lock`;
   - concurrent first openers either join the same initialization or receive a
     typed conflict without running migrations twice.
6. Treat the bootstrap metadata row as the bootstrap operation intent. It
   contains a random bootstrap ID, family, generation, genesis, namespace, and
   `initializing`/`ready` state before normal mutation-intent tables exist.
7. Replace the pooled migration executor with a connection-bound executor:
   - accept caller context and run lock waits, migration bodies, version
     transitions, rollback, and lock release through contextual operations;
     never call a non-contextual `m.Up()`;
   - SQLite commits the durable `initializing` marker, then applies every
     pending non-idempotent DDL body, migration version transition, foundation
     table, and final `ready` transition in a second (final) pinned
     `BEGIN IMMEDIATE` transaction;
   - PostgreSQL keeps the advisory lock and executes each migration body plus
     version transition atomically, then atomically creates foundation state and
     marks `ready`;
    - no wallet mutation is admitted while bootstrap is `initializing`.
    In PR 1's intermediate build, the caller is an explicit bounded Manager
    lifecycle context derived from Manager shutdown plus configured migration
    timeout, never `context.Background`. PR 3 removes that temporary internal
    adapter when context-bearing public Create/Load and Controller arbitration
    land; its request context then reaches this same executor unchanged.
8. Reconcile an ambiguous bootstrap result under the same lock. Exact `ready`
   means success; `initializing` resumes from the last clean committed version;
   absent metadata permits fresh bootstrap; any family/genesis/namespace
   mismatch fails closed. Never replay DDL that committed without its version
   transition.
   Cancellation before COMMIT begins rolls back and releases every bootstrap
   resource. Once COMMIT begins, a bounded non-canceled context reconciles the
   result: exact `ready` wins as success, a clean rolled-back state returns the
   caller's cancellation, and an unprovable state returns public ambiguity.
9. Bind to the full genesis hash, not only network magic or coin type.
10. Do not build pre-marker adoption machinery. The role route has not shipped,
   so an old default migration table, old role tables, or any non-empty
   unmarked namespace fails closed with a recreate instruction.
11. Inspect PostgreSQL's ambient namespace before initializing the dedicated
   schema. Never create an empty wallet beside a pre-marker or foreign wallet.
12. Expose stable public errors for incompatible schema family, unsupported
   generation, dirty migration state, and wrong network without including a
   plaintext PostgreSQL DSN.
13. Record the schema-family, migration-locking, PostgreSQL namespace,
    genesis-binding, and fail-closed pre-marker decision in a new ADR before
    implementation lands.
14. Create one persistent SQLite DB-adjacent control namespace only after
    preflight classifies an empty namespace as initializable. Store one random
    namespace ID in its metadata and the database identity row. Open control
    files without following symlinks; require regular, singly linked files and
    stable device/inode identity. Never unlink or replace them on close, crash,
    takeover, or repair.
15. Require a local filesystem with proven cross-process advisory-lock and
    release-on-process-death semantics. Recheck canonical path, file identity,
    link count, namespace ID, and database identity under the lock before any
    driver open; otherwise fail closed with a typed unsupported-filesystem
    error. This protects accidental aliases and protocol-following processes,
    not hostile same-principal path replacement after the final recheck.
    Require safe principal ownership/modes for the database parent and its
    rename-controlling ancestors. A hostile-path guarantee would require a
    separately approved handle-relative writable SQLite VFS.
16. Freeze migration bodies 1 through 11 when PR 1 can publish a marked Store.
    Check their SHA-256 manifest and never edit, reorder, or rename an applied
    migration. Every later schema change uses a new paired forward migration:
    - PR 3: `000012_runtime_foundation`;
    - PR 4: `000013_script_protection`;
    - PR 5: `000014_canonical_chain`;
    - PR 6: `000015_accounts_v2` and `000016_discovery_allocation`;
    - PR 7: `000017_terminal_invalidation` and
      `000018_publication_obligations`.
    Migration numbers are one strict stack: PR 3 < PR 4 < PR 5 < PR 6 < PR 7.
    PR 6 lands migration 15 before 16 in ordered commits; PR 7 lands 17 before
    18. Every migration-bearing PR contains all earlier numbered migrations. CI
    upgrades PR-1 fixtures through every intermediate version on both backends,
    and no
    independently mergeable branch may claim the same next number.
    The enumerated owner/version order is binding even before an individual
    migration freezes. An out-of-order-ready PR waits and is not renumbered;
    changing the order requires an explicit revision of the complete migration,
    fixture, PR-dependency, CI, and evidence map before any affected PR becomes
    independently usable.
    Allocate another version for newly discovered schema work; never fold it
    into an earlier migration.
    This intentionally replaces this feature line's edit-in-place convention.
    Before PR 1, no durable family marker identifies a usable role database, so
    unreleased migrations may be corrected. PR 1 is the freeze because it first
    publishes marked Stores and upgrade fixtures that stacked PRs/reviewers may
    consume before release; mutable DDL after that point would invalidate crash
    evidence and diverge already marked databases.
17. Hold one Store-lifetime shared schema lease from locked preflight until all
    data handles close. Bootstrap/migration takes it exclusively. A Store is not
    published without the shared lease, and migration cannot begin while an old
    Store retains it. After migration, transition exclusive-to-shared without
    an unlocked gap and recheck family, generation, dirty state, and version.
    SQLite uses the persistent control lock; PostgreSQL uses a dedicated
    session advisory-lock connection outside `MaxConnections`. Close drains
    Store calls and data handles before releasing the lease; uncertain close
    retains it until process death.
18. Fence PostgreSQL data transactions against lifetime-session loss. Record the
    lease backend PID/generation and a Store-random session guard. The lifetime
    session also holds a second advisory lock derived from lock kind
    `lease-guard` plus that random guard. Every data transaction first takes the
    transaction-scoped shared schema lock and verifies through server lock state
    that the recorded backend PID simultaneously owns both the fixed shared
    schema lease and exact random guard lock, then verifies family/version/
    generation and the local lease latch.
    A watcher closes new admission when the dedicated session dies; the Store
    never reacquires transparently. Already-fenced transactions may finish, but
    later reads/writes fail until reopen. Migration advances the generation.
19. Require PostgreSQL connection mode `direct` or `session-pool`. Before any
    session lock, reject transaction/statement PgBouncer with a typed redacted
    error, query exposed pooler mode, and probe affinity across transactions via
    `pg_backend_pid()` plus a random advisory lock. Hidden/lying poolers are
    unsupported. Include mode in runtime option conflicts; alias support covers
    session mode only.
20. Bind SQLite control identity to the driver-open target. Remove Manager-side
    preflight `MkdirAll`; use a correctly escaped `file:` URI with `net/url`
    path/query encoding, never path-plus-`?`. Existing-file inspection is
    explicit side-effect-free read-only. After initializable classification
    under the
    control lock, create directories handle-relatively without symlink traversal
    at 0700 and a new file exclusively/no-follow at 0600. On the pinned RW
    connection, before mutating pragmas, require `PRAGMA database_list` main to
    resolve to canonical basename and the locked regular file's device/inode/
    link identity. Registry, preflight, lock, and driver target must match.
21. Before each forward migration whose prior schema has runtime foundations,
    hold the exclusive schema lease and inspect health/intents using a frozen
    prior-version reader. Reconcile every `applied` intent under its old schema;
    quarantined state must be repaired before DDL. An `awaiting_side_effect`
    intent crosses only when the migration explicitly preserves its payload,
    replay action, identity, and proof contract. Otherwise return typed
    maintenance-required without changing schema/version. Every migration
    13-18 declares its old-state reader and evidence-preservation rule; never
    transform unresolved candidate facts before their old-version verifier runs.

#### Primary Files

- `wallet/internal/sql/sqlite/migrations.go`
- `wallet/internal/sql/pg/migrations.go`
- `wallet/internal/sql/{sqlite,pg}/migrations/`
- `wallet/internal/db/{sqlite,pg}/store.go`
- `wallet/runtime_store_registry.go`
- the new database-identity ADR under `docs/developer/adr/`
- SQL backend configuration and open tests

#### Tests

- Empty database initialization.
- Reopen with matching family, generation, and genesis.
- Salvage/foreign schema with the same migration version.
- Non-empty unmarked schema.
- Dirty and newer migration versions.
- Wrong genesis on a fresh process after closing the first Store.
- Concurrent matching and conflicting first opens through independent pools.
- A separate-process SQLite race proves bootstrap is database-locked.
- PostgreSQL coexistence with lnd tables and an unrelated migration table.
- A populated ambient PostgreSQL wallet cannot be shadowed by an empty new
  dedicated schema.
- Pre-marker SQLite and PostgreSQL fixtures fail with a recreate instruction
  and remain byte-for-byte/catalog unchanged. SQLite creates no WAL, SHM,
  journal, migration table, or side file.
- Interrupted `initializing` metadata resumes or fails deterministically.
- Frozen SQLite/PostgreSQL fixtures upgrade one version at a time and directly
  from version 11 through the latest migration. Include populated account rows,
  sparse block history, interrupted transitions, checksum mismatch, and
  latest-versus-upgraded schema equivalence.
- Ambiguous committed and rolled-back bootstrap outcomes reconcile through the
  bootstrap ID before any wallet mutation.
- Crash injection after marker creation, dirty/version state, DDL execution,
  version cleanup, foundation creation, `ready`, and final COMMIT
  acknowledgement resumes from the last clean version on both backends.
- A race replacing an acceptable SQLite file before locked recheck is rejected
  without mutating the replacement.
- Cancellation during lock wait, migration SQL, version transition, foundation
  creation, and final acknowledgement releases locks and connections and
  returns the reconciled durable outcome.
- SQLite directory/file symlink aliases, hard links, control-file replacement,
  and an unsupported-filesystem seam fail closed. Persistent control identity
  survives clean close and crash takeover.
- Process A's shared schema lease blocks process B's migration and old-schema
  write. Cancellation, clean close, and crash release resources in the defined
  order; PostgreSQL data-pool size one remains usable.
- PostgreSQL bootstrap/migration, lifetime lease, and transaction fence derive
  the identical signed schema key across DSN/pool aliases and contend by lock
  mode; wallet/session/create IDs cannot change it. Other lock kinds include
  only their declared durable scope identity.
- Terminate only PostgreSQL's lifetime lease PID while data connections remain
  healthy. One already-fenced transaction may finish; every later read/write
  fails before another process migrates, and the old Store stays fenced.
- Reuse/stub the recorded PID and mismatch the random guard lock; data
  transactions reject both cases even when the fixed schema lock appears held.
- Upgrade fixtures with old-version applied intents reconcile before DDL;
  quarantine returns maintenance-required unchanged. Awaiting-side-effect
  fixtures cross only migrations declaring exact replay/evidence preservation.
- Direct and real session-mode PgBouncer pass the affinity probe. Real
  transaction/statement modes fail before migration and leave no advisory lock;
  errors/logs contain no DSN or credentials.
- Literal `?`, `#`, `%`, space, and UTF-8 filenames cannot inject URI options.
  Replacement before recheck or between driver open/database-list verification
  fails; foreign rejection creates no directories/files/sidecars. Secure create
  rejects symlink/unsafe components and proves 0700/0600 modes and inode match.
- Unsafe ancestor ownership/mode fails before driver open. Do not claim that the
  pathname driver rejects hostile same-principal replacement after recheck.
- Error text and logs contain no credentials or raw DSN.

#### Exit Gate

- Schema compatibility and network identity are checked before any wallet
  mutation.
- PostgreSQL wallet tables and migrations have an explicit namespace.
- A foreign or wrong-network database cannot reach ordinary wallet queries.

## BW1: Public Contract And Recovery

### BWR-001: Public Role Types And Errors

#### Issue And Rationale

`wallet.AccountManager` returns `wallet/internal/db.AccountInfo`. An external
consumer cannot name that type or implement the interface because Go forbids
imports across the `internal` boundary. `wallet.TxStatus` is currently an alias
of the internal DB enum, so its ownership is also accidental rather than an
explicit wallet contract.

Some Manager, account, address, lifecycle, and signer paths still expose
backend-specific errors. This forces callers either to compare text or to
import implementation details. A role interface is not public in practice when
its values and control-flow errors are internal.

#### Detailed Fix

1. Define wallet-owned `AccountID uint64` and `AccountSelector{Scope,
   AccountID}`, and expose the selector from `AccountInfo`. Zero is valid for
   KV; SQL IDs must be positive and at most `MaxInt64`; KV IDs must fit
   `uint32`; conversions never truncate. Widen every stable account-identity
   carrier while BIP32 account numbers remain optional `uint32`. Define closed
   rescan targets `AccountSelector` and wallet-wide `RawImportSelector` for
   accountless imports. Bulk allocation accepts only `AccountSelector`.
2. Define a real wallet `TxStatus` type and constants in
   `wallet/tx_reader.go`; do not use a type alias. This intentionally supersedes
   the rewritten stack's recent internal-type re-export.
3. Convert Store account and status values at the wallet boundary.
4. Preserve optional identity in public results. In particular,
   `AccountInfo` and derived-address metadata distinguish an absent master
   fingerprint from a present fingerprint whose value is zero.
5. Inventory existing public wallet errors before adding anything. Reuse
   existing wallet sentinels for lease and transaction lookup behavior.
6. Add only the missing public errors required for wallet not-found/existing,
   account and scope conflicts, address/key not found, watch-only/private-key
   unavailability, lifecycle conflicts, unsupported operations, Store
   ambiguity, unhealthy durable state, and a wallet-owned `ErrWalletLocked`.
7. Translate internal errors once at the role boundary. Wrap the wallet
   sentinel for `errors.Is`; include the internal error only as non-wrapping
   context so callers cannot accidentally depend on it.
   In particular, translate `keyvault.ErrVaultLocked` and legacy locked errors
   to `ErrWalletLocked`; never wrap the internal sentinel as an `errors.Is`
   target.
8. Pass `context.Canceled` and `context.DeadlineExceeded` through before wallet
   error translation only when cancellation wins the operation arbitration, so
   standard `errors.Is` matching is never erased.
9. Carry caller context through Controller request messages and use a
   `queued -> claimed -> finished` state machine. Cancellation winning while
   queued prevents dispatch. Unlock remains reversible until completion wins;
   a cancellation winner clears newly opened vault state. Lock and passphrase
   rotation become irreversible when claimed, after which the completed or
   reconciled durable result wins over caller/lifecycle cancellation. SQL
   rotation reconciles ambiguous COMMIT with a bounded internal context, while
   legacy walletdb returns its definitive transaction result.
   Add non-mutating `Vault.Authenticate(ctx, passphrase)` for repeat Unlock. An
   already-unlocked wallet authenticates without Lock/Unlock or runtime-key
   replacement. Failure/cancellation preserves prior auth and the original
   timer deadline; only a completion winner drains/resets/disables the timer.
   SQL zeroes temporary key material, and KV adds a non-destructive waddrmgr
   passphrase check. Initial-unlock cancellation may compensate by locking.
10. Keep `AddressInfo` public, but correct its imported provenance as part of
   BW5: `Imported` comes from Store provenance, not from absence of a derivation
   path.
11. Add compile-time assertions for every role implemented by `*wallet.Wallet`.
12. Add an external-package test that defines and mocks the complete role
    composition without importing `wallet/internal/db`.
13. Update ADR 0002 with queued/claimed/finished arbitration and the reversible
    Unlock versus irreversible Lock/passphrase rules.
14. Replace Manager's contextless APIs with `Create(ctx, cfg, params)` and
    `Load(ctx, cfg)`. Thread the request context through per-name guards,
    teardown waits, Store creation/open, registry acquisition, preflight,
    schema-lease waits, migration, initial-account/completion reconciliation,
    and final acknowledgement. Do not preserve background-context wrappers
    without a concrete external caller. Derive Wallet lifetime independently so
    a successful Wallet survives request cancellation.
    Land this replacement with PR 3's Controller/Store transport, not PR 1's
    type-only contract. SQL passes context to acquisition/queries until COMMIT,
    then reconciles with a bounded internal context. KV checks cancellation
    before claiming each walletdb transaction; after `Update` starts its
    definitive result wins and no cancellation goroutine races the callback.
15. Carry `AccountID` through every recovery/rescan carrier: target/filter DTOs,
    per-account maps, branch horizons, job manifests, callbacks, Store queries,
    registration payloads, and result metadata. Keep optional BIP32 account
    number separately typed `*uint32`; never cast an identity to derive a path.
    Narrow only at the checked KV adapter boundary. Add a static denylist for
    `uint32` identity fields in wallet-owned recovery types.

#### Primary Files

- `wallet/account_manager.go`
- `wallet/address_manager.go`
- `wallet/tx_reader.go`
- `wallet/controller.go`
- `wallet/interface.go`
- `wallet/wallet.go` and the final wallet-owned error location selected in S0
- `docs/developer/adr/0002-controller-syncer-architecture.md`

#### Tests

- External-package compile and mock test.
- Conversion tests for every account and transaction status.
- `errors.Is` tests at Manager and each changed role boundary.
- External-package `errors.Is(err, wallet.ErrWalletLocked)` succeeds for SQL
  locked script/signing reads; internal keyvault/waddrmgr sentinels do not.
- Cancellation before dispatch, during lifecycle work, deadline expiry, and
  response/cancel races; canceled unlock leaves no opened vault state.
- SQL committed/rolled-back passphrase-rotation cancellation races, legacy
  walletdb rotation races, and Lock response/cancel races.
- Already-unlocked wrong/canceled authentication preserves prior state and timer
  deadline. Successful repeat renews/disables the timer without stale expiry;
  use an injectable clock across SQL, spendable KV, and SQL watch-only vectors.
- Manager cancellation while waiting for create/load/teardown, migration,
  initial accounts, and completion leaks no registry reference. Durable outcome
  wins once COMMIT begins; a successful Wallet outlives the request context.
- External code builds a `RescanTarget` and bulk selector directly from returned
  `AccountInfo` without importing an internal package.
- IDs above `MaxUint32` survive SQL get/list/rescan/allocation through
  `MaxInt64`; overflow fails before SQL, KV rejects out-of-range IDs, and a real
  ID equal to the old imported sentinel is not treated as raw import.
- Recovery jobs for SQL IDs above `MaxUint32` preserve target, horizon,
  registration, restart, and result ownership exactly; KV rejects before enqueue
  and no truncating alias can scan another account.
- `RawImportSelector` scans only accountless imports and is statically
  ineligible for bulk allocation; external code constructs both target kinds.
- A denylist test that rejects `wallet/internal/db` in public role signatures.

#### Exit Gate

- An external package can name, use, and mock every selected role.
- No modern role signature exposes an internal type.
- No lnd-specific method is added to the legacy `wallet.Interface`.

### BWR-002: Recovery-Window Validation

#### Issue And Rationale

`Config.validate` rejects every `RecoveryWindow` below
`MinRecoveryWindow == 20`. Recovery code supports zero and small positive
lookaheads, while lnd uses zero normally and smaller values in tests. Rounding
or rejecting those values changes caller policy without a safety rationale.

#### Detailed Fix

1. Remove the artificial minimum check from `wallet.Config.validate`.
2. Define `MaxRecoveryWindow = 100_000` as the supported resource bound. Accept
   every value from zero through that limit exactly and reject larger values
   with a stable public error.
3. Zero means no undiscovered-address lookahead. It does not disable ordinary
   synchronization of persisted addresses, scripts, or watched outpoints.
4. Use checked arithmetic for `nextIndex + recoveryWindow`. Before deriving,
   verify that the remaining non-hardened child range can hold the complete
   requested window; never wrap or enter the hardened range.
5. Remove or rename `MinRecoveryWindow`; a recommended default must not be
   represented as a validity bound.
6. Keep rejection of negative signed RPC values in lnd before conversion. No
   signed field is added to btcwallet.

#### Primary Files

- `wallet/wallet.go`
- `wallet/wallet_test.go`
- `wallet/recovery.go`
- tests that use `MinRecoveryWindow` only to satisfy validation

#### Exit Gate

- Values 0, 1, 2, 19, and 20 are accepted unchanged.
- `MaxRecoveryWindow` is accepted, and `MaxRecoveryWindow+1` is rejected.
- A horizon near `waddrmgr.MaxAddressesPerAccount` fails instead of wrapping.
- Zero continues syncing persisted targets without deriving lookahead targets.
- Existing recovery and no-recovery vectors remain green.

## BW2: Vault And Watch-Only Correctness

### BWR-003: Semantic Script Protection And SQL Conformance

#### Issue And Rationale

The earlier production-defect premise is stale. `WalletVault` maps `CKTPublic`
to the passphrase-derived script key used for `CKTScript`, so an unlocked SQL
watch-only wallet can import and reopen a tapscript. The current
production-vault SQLite round trip proves that behavior.

The remaining problem is architectural. Wallet code still chooses legacy
`waddrmgr.CryptoKeyType` values even though SQL has no separate always-available
public crypto key. KV can read a non-secret script while locked, whereas SQL
currently surfaces internal `keyvault.ErrVaultLocked` until its script key is
opened; the role boundary must translate it to public `wallet.ErrWalletLocked`.
Manager-created restart and PostgreSQL parity also remain unproven.

#### Detailed Fix

1. Add semantic script seal/open operations to
   `wallet/internal/keyvault.Vault`.
2. Make SQL `WalletVault` use the single-passphrase script key and keep Store
   data opaque.
3. Separate new-seal policy from opening persisted data. `SealScript` receives
   the intended protection class, while `OpenScript` receives a versioned
   protection descriptor or self-describing envelope persisted with that exact
   record.
4. Make `LegacyManagerVault` choose `CKTPublic` or `CKTScript` from the
   persisted per-record descriptor when opening existing data, never from
   wallet mode.
   Spendable legacy wallets may contain either public or secret scripts.
5. Route `ImportTaprootScript`, `ScriptForOutput`, and signer script lookup
   through the semantic vault operation.
6. Remove wallet-domain branching on `waddrmgr.CryptoKeyType` where the branch
   exists only to select script encryption.
7. Retain the current production-vault SQLite round trip as a positive
   regression. Add Manager-created SQLite/PostgreSQL stop/load/restart vectors
   using the real `WalletVault`; do not describe a hybrid pairing as hiding a
   current failure.
8. Preserve/document lock semantics: SQL public roles return `ErrWalletLocked`
   until unlocked, while legacy KV may open a non-secret script through its
   always-available public key.
9. Add `000013_script_protection` before semantic readers. Backfill SQL rows as
   legacy-v0 with an outer version/class/binding descriptor using the frozen old
   wallet-mode rule only in migration. On authenticated unlock, resumably
   decrypt/reseal outside Store transactions into v1 envelopes whose
   authenticated plaintext binds version, protection class, record identity,
   and script bytes; compare outer/inner descriptors. CAS original ciphertext,
   reconcile exact candidates, and persist no plaintext. Starting-owner may
   build/register the watch snapshot from plaintext output scripts while v0
   encrypted envelopes remain, so Start can become active. Block only encrypted
   script opening/signing until post-Start authenticated Unlock completes the
   conversion. New SQL/KV seals use v1; ordinary reads never infer class from
   mode.

#### Primary Files

- `wallet/address_manager.go`
- `wallet/signer.go`
- `wallet/internal/keyvault/vault_cipher.go`
- `wallet/internal/db/sqlite/addressstore_common.go`
- `wallet/internal/db/pg/addressstore_common.go`
- `wallet/internal/sql/{sqlite,pg}/migrations/000013_script_protection.*.sql`
- script-envelope conversion/CAS queries for both SQL backends
- `wallet/signer_test.go`

#### Tests

- Current unlocked SQLite production-vault round trip remains green.
- Manager-created spendable/watch-only imports survive stop/load/unlock/restart
  on SQLite and PostgreSQL.
- Locked SQL `ScriptForOutput` returns public `ErrWalletLocked`; unlock restores
  access
  without changing persisted script identity.
- KV keeps non-secret script access through its public-key path while secret
  scripts require the script key.
- Wrong passphrase, corrupted ciphertext, and malformed/unknown protection
  descriptors fail closed.
- Legacy spendable/public, spendable/secret, and watch-only/public fixtures open
  with their persisted protection descriptor.
- PR-1 SQLite/PostgreSQL fixtures upgrade through 12/13/latest; unlock
  conversion survives batch crash/ambiguity without ciphertext drift.
  Version/class/record substitution fails authentication; locked state remains
  locked.
- An upgraded wallet with v0 rows starts from plaintext watch scripts, remains
  unable to open/sign encrypted scripts, then authenticated Unlock converts
  resumably and survives restart entirely on v1.

#### Exit Gate

- Wallet code no longer chooses a legacy script cipher type.
- Manager-created watch-only tapscripts survive restart on both SQL backends
  without regressing current unlocked SQLite behavior.
- SQL and KV locked-read semantics are explicit and tested.
- No plaintext script secret is placed in Store metadata or logs.

### BWR-004: Explicit Watch-Only Lifecycle Semantics

#### Issue And Rationale

SQL watch-only `Unlock` succeeds because the vault may need to open protected
script material. KV delegates to waddrmgr and returns `ErrWatchingOnly`. A
Controller caller therefore receives different lifecycle behavior for the same
wallet-level mode.

The difference cannot be erased by swallowing `ErrWatchingOnly`. Legacy
waddrmgr has no private-passphrase verifier for a watch-only manager, reports
itself locked, and keeps its public script key available. Pretending that it
authenticated would create state that does not correspond to a durable
credential or key transition.

The fresh modern lnd path in this milestone is SQL-only. It needs precise SQL
watch-only behavior and a stable public result for unsupported KV behavior, not
manufactured parity.

#### Detailed Fix

1. Define SQL Controller semantics explicitly: watch-only `Unlock` authenticates
   the wallet passphrase and opens protected script material; `Lock` clears that
   material.
2. SQL watch-only unlock never loads private account or
   address keys.
3. Keep signing and private derivation guarded by a public wallet-owned
   watch-only/private-key-unavailable error.
4. Make `IsLocked` and Controller auth state agree after SQL watch-only lock and
   unlock operations.
5. Preserve legacy KV behavior with one stable contract: watch-only `Unlock`
   returns public `ErrUnsupportedOperation`; watch-only `Lock` is idempotent
   success. Controller authentication remains locked and `IsLocked` remains true
   after either call. Do not add an adapter-only authentication boolean.
6. Treat unsupported public-passphrase behavior separately. Do not leak
   keyvault or waddrmgr errors as accidental API semantics.

#### Primary Files

- `wallet/controller.go`
- `wallet/state.go`
- `wallet/internal/db/kvdb/vault.go`
- `waddrmgr/manager.go` for behavior verification only; preserve its legacy
  semantics

#### Exit Gate

- SQL watch-only authentication and script-key lifetime are explicit.
- Watch-only signing still fails without touching a local private key.
- SQLite and PostgreSQL agree; KV Unlock returns public unsupported, Lock
  succeeds idempotently, and Controller/`IsLocked` remain locked.

## BW3: Atomic Live Chain State

### BWR-005: Synced Tip Before Transaction State

#### Issue And Rationale

The comparative test observed wallet height at the miner tip before it observed
the mined transaction and confirmed balance. btcd emits `BlockConnected` and
`RelevantTx` separately. The syncer currently can persist the connected tip
before the relevant transaction notification is applied.

This may be a consistency window rather than a permanently missed
confirmation, but it violates the useful invariant that `SyncedTo == h` means
wallet transaction, UTXO, and balance state through block `h` is durable and
queryable.

#### Detailed Fix

1. Keep the shared chain notification stream and legacy/KV consumers unchanged.
   Canonical coalescing belongs to the modern syncer contract, not a global
   suppression or reinterpretation of `BlockConnected`.
2. Persist a wallet watch generation. Increment it atomically whenever a
   watch-chain address, recovery horizon/lookahead, raw watched script, or exact
   watched outpoint script changes. Key-only accounts never affect it. Bind the
   effective recovery-lookahead policy to the active runtime epoch.
3. Add one coherent Store watch-seed snapshot operation. SQLite uses one read
   transaction and PostgreSQL uses one read-only repeatable snapshot. Return the
   account/horizon state, exact raw scripts, watched `(outpoint, script)` pairs,
   watch generation, runtime epoch, and policy identity. Digest the sorted
   scripts, outpoints, horizons, discovery modes, effective lookahead,
   generation, and runtime epoch.
   Include stable account identity, XPub/schema, expected branch counters, and
   the database chain-view epoch needed to prepare recovery horizons.
4. Make backend events wakeups/hints, not completeness proof. btcd and bitcoind
   may fetch and exactly filter the full block. Neutrino evaluates the
   authenticated BIP158 basic filter against the exact script snapshot, advances
   on an authoritative non-match at generation `G`, and fetches the full block
   only on a match. Missing, invalid, or unauthenticated filter evidence cannot
   advance the tip. Preserve the legacy address-oriented `FilterBlocks`
   contract and add a modern exact-script proof path.
5. Carry the snapshot digest and generation `G` into the canonical Store batch.
   Compare both generation and active runtime epoch inside the transaction that
   applies effects and advances the tip. A mismatch discards the candidate and
   rebuilds the coherent snapshot before exposing the new tip.
6. Replace address-only watch inputs with exact script-capable targets so bare
   multisig and outputs that have no single address are never dropped.
7. Handle btcd notifications by block state:
   - `RelevantTx.Block == nil` applies immediately as unmined and never enters a
     canonical correlation buffer;
   - confirmed notifications buffer only by exact chain-view epoch, height, and
     block hash and are optimization hints for that matching block;
   - reconnect, overflow, ordering violation, or unmatched block clears the
     buffer and falls back to the backend-specific authoritative proof.
8. Carry complete ancestry into the modern batch. The canonical event or
   coalesced result includes the block header/previous hash, not only height and
   block hash.
9. Use distinct canonical and targeted Store DTOs. Canonical batches carry the
   expected tip, contiguous ancestry, snapshot digest/generation, and canonical
   identity and may advance `SyncedTo`. Targeted batches carry job ID, expected
   database chain-view epoch, and each confirming block's exact height, hash,
   and previous hash. The applying transaction requires every confirmation to
   match
   an existing canonical block row. Targeted mode never inserts a block, calls
   `ensureBlockExists`, or changes `SyncedTo`; missing, wrong, pruned, or stale
   membership rolls back with a typed result.
10. Extend canonical Store DTOs with the expected current `(height, hash)` and
    the candidate block's previous hash. Compare those facts inside the same
    Store transaction that applies transactions and advances the tip; a
    syncer-side pre-read is not sufficient.
11. Apply transaction effects and advance the sync tip in one Store operation,
   reusing `ApplyTxBatch` or `ApplyScanBatch` rather than inventing another
   transaction-ingestion path.
12. Keep the syncer's chain loop as the serialization owner and reject any
    parameter combination that mixes targeted mode with a tip update.
13. Compare the expected current tip in the atomic canonical batch. Forward
    progress requires contiguous ancestry: the new height is current height plus
    one and the new header's previous hash equals the current hash.
14. Send a canonical gap to catch-up rather than jumping the synced watermark.
    A stale canonical replay may no-op only when its durable intent/digest
    proves the complete batch was applied. Matching candidate rows alone is
    insufficient. A targeted historical addition beneath a newer tip is normal.
15. On reconnect or detected chain-view change, mark unsynced, increment the
    durable chain-view epoch under exclusive admission, clear correlation
    state, wait for backend currentness, compare best height/hash, roll back to
    the common ancestor, rebuild/register the complete watch snapshot, catch
    up, then admit live events. A targeted batch racing rollback either commits
    first and is detached by rollback, or observes the new epoch and restarts.
16. Poll best `(height, hash)` while idle so bitcoind/neutrino same-height
    reorgs are detected without a reliable reconnect notification. Never
    declare synced from height equality alone.
17. Keep rollback/reorg as the explicit operation allowed to lower the tip.
18. Finish ambiguous reconciliation before the chain loop processes another
    block event. Replay the complete batch only when candidate and expected-tip
    checks prove it safe.
19. Update ADR 0003 with compact-filter generation proofs and ADR 0004 with the
    existing-canonical-membership and never-create-targeted-block invariants.
20. Normalize duplicate account/branch horizons, derive every horizon child and
    perform KV managed-address encryption outside the write transaction, then
    pass prepared horizon DTOs into `ApplyScanBatch`. The write CASes every
    expected counter/generation with transaction effects and tip movement. One
    stale horizon aborts the entire batch; discard/reprepare without partial
    effects. KV stops publishing caches before commit.
21. Add one singleton database canonical frontier with height/hash,
    previous-hash ancestry, monotonic database chain-view epoch, exact digest,
    arbitration
    generation, and `ready`/`arbitrating` state. Canonical extension serializes
    this row and compares expected epoch/frontier atomically. Wallet-local tips
    never arbitrate shared block identity.
22. A conflicting occupied-height hash returns a typed canonical-view-changed
    result, then routes through exclusive database chain admission. Increment
    the epoch, mark all wallets unsynced, preserve the old frontier while active
    backends disagree, and adopt one contiguous common-ancestor result only
    after agreement. Crashed arbitration is replaceable. Targeted/ordinary
    ingestion never creates canonical rows and old-epoch work restarts.
23. Preserve legacy `chain.Interface`, but add a context-aware modern canonical
    proof interface for best tip, hashes, headers, blocks, compact filters, and
    batches. Use caller cancellation plus a hard bound for birthday, polling,
    rollback, canonical filtering, and targeted proof. Backend cancellation must
    terminate/track pending work without detached goroutine leaks. Reset and
    shutdown cancel the run before waiting; dependency support lands/tags before
    PR 5 if btcd or neutrino cannot cancel requests.
24. Update ADRs 0001 and 0006 to state that wallet transaction state is scoped
    per wallet while canonical ancestry and chain-view epoch are database-wide.
25. Add forward migration `000014_canonical_chain` for previous-hash ancestry
    and reset evidence. PR 3's migration 12 already owns the singleton canonical
    frontier/epoch/arbitration foundation.
26. Define canonical `extend` and `attach` modes. Extend advances the ready
    database frontier by one. Attach verifies an existing ready canonical row at
    the target wallet's expected tip plus one, then atomically applies only that
    wallet's transaction/UTXO/balance/horizon/tip effects without changing the
    frontier. Persist a retained wallet application proof keyed by wallet,
    chain-view epoch, and height with block/batch/snapshot digests and operation
    ID, including empty batches. Another wallet's proof never proves this one;
    replay/reconciliation requires the caller's operation ID and retained proof.
    Keep ready-epoch proofs; compact old epochs only after originating sessions
    are fenced and reconciliation watermarks permit it.
27. Make arbitration agreement durable. A required participant is `(wallet,
    runtime epoch, owner session, backend kind)` tied to live starting/active
    ownership. Outside arbitration, graceful Stop retires it. During a frozen
    generation, Stop marks itself stopping and releases ownership but cannot
    remove itself; it remains required until the arbiter independently acquires
    its abandoned lock and fences its freshness obligation. The retained member
    row and any dissent remain generation facts; disagreement cannot withdraw.
    Arbitration freezes the participant set, old frontier, conflict, generation,
    proof round, and arbiter owner while blocking ingestion. Each
    participant writes only its fresh epoch-bound best-tip/common-ancestor/
    contiguous-ancestry digest as an `ACK(prefix)` or durable `NACK(reason)`.
    Adoption requires at least one fresh non-fenced participant, unanimous fresh
    ACKs from every non-fenced required participant on the highest common
    prefix not above the minimum fresh height, and no NACK recorded by any
    member of the frozen generation, including a later-fenced member. Expiry
    requires refresh,
    not eviction. Abandoned-lock fencing removes only that participant's
    freshness obligation; its retained row and dissent remain generation facts.
    An all-fenced generation cannot adopt. A NACK is monotonic for the round:
    cancellation, Stop, takeover, and a later ACK cannot erase it. Stop records
    participant-stopped and releases serving, but cannot synthesize agreement or
    withdraw dissent. Takeover CASes arbiter ownership and reuses the same
    generation, set, dissent, and proofs.
    Persist exact phases `PREPARED` (frozen set/digests), `COLLECTING`,
    `DECIDED` (unanimous prefix), `APPLYING`, and terminal `READY` or `FAILED`;
    only the longest complete prefix is resumable and no row publication
    precedes READY.
    Every row is arbiter-owner/epoch fenced and phase transitions CAS the prior
    digest. Freeze `proof_ttl` per generation: default 30 seconds, range 5
    seconds through 5 minutes. PostgreSQL proof/decision age uses server time on
    the pinned session and accepts only `decision_time < expires_at`. SQLite
    uses a monotonic deadline while the lock owner lives; restart/takeover
    invalidates freshness and requires every proof to refresh, so wall-clock
    skew cannot create early adoption. Never compare participant host clocks.
28. Migration 12 creates canonical state as `non_authoritative`. On a fresh
    database with no wallet/non-genesis state, migration 14 deterministically
    installs the bound genesis as `ready`; this is initialization, not empty-set
    arbitration. An upgraded version-12 database enters `cutover_required`;
    migration is DDL/control-state only and performs no chain RPC. After Store
    open, admit only Manager Load, Start-to-starting ownership, the named
    `cutover-owner`, and maintenance diagnosis/repair while no cutover owner is
    held; ordinary roles stay blocked. Cutover claim requires healthy shared
    state and no applied, quarantined, or otherwise unresolved shared-state
    intent.
    The coordinator freezes a nonempty set of registered starting participants,
    enters `cutting_over`, increments epoch, marks wallets unsynced, resets
    post-genesis effects, and performs bounded chain proof outside Store
    transactions. One adoption transaction verifies unanimous fresh proofs,
    installs ancestry, reconciles confirmations, and sets `ready`. Crash
    takeover resumes the durable generation/set/proofs. Empty-participant
    upgrade adoption and maximum-row/tip inference are forbidden; wallets
    start/catch up after ready.
    An unprovable cutover enters failed/quarantined state and releases or fences
    cutover ownership so maintenance can acquire authority. Verified repair
    preserves the generation evidence and permits a fenced resumption.
29. Bind each targeted job to one coherently read ready canonical ceiling
    `(epoch, height, hash, digest)`. Scan only existing canonical rows from the
    requested start through that ceiling, never backend best height. Start above
    the ceiling returns a typed no-write error; unready/arbitrating returns
    typed chain-view state. Every batch carries the ceiling; epoch change
    aborts/restarts. Canonical notifications queued during the scan remain
    outside its range and run afterward.
30. Expose one context-first `RunTargetedScan(ctx, TargetedScanRequest)` on the
    Wallet/Manager boundary; internal callers cannot bypass it. The request
    carries wallet ID, start, exact target snapshot/digest, expected chain-view
    epoch, runtime/backend generation, and configured hard bound. The
    coordinator atomically captures and returns the canonical ceiling and a
    typed result: `complete`, `no_range`, `stale_epoch`, `retryable_backend`,
    `unsupported`, or `not_attempted`; error preserves cause while disposition
    alone drives retry. Caller cancellation plus hard deadline aborts backend
    and Store work, waits for workers, and guarantees no writes beyond a
    committed complete batch.
31. Any ordinary/targeted scan batch that commits new derived/imported watch
    material, branch horizon, or relevant outpoint also writes the exact
    generation-bound registration event in that same Store transaction. A stale
    batch writes neither. The active owner drains it before declaring the scan
    complete/readiness true; without an owner it remains queued for startup.
    Event identity/digest follows the logical registration protocol, so crash,
    replay, and successor ownership cannot lose or duplicate the watch update.

#### Primary Files

- `wallet/syncer.go`
- `chain/interface.go`
- `chain/btcd.go`
- `chain/bitcoind_client.go`
- the neutrino filtered-block adapter
- `chain/block_filterer.go`
- `wallet/recovery.go`
- `wallet/internal/db/data_types.go`
- `wallet/internal/db/{sqlite,pg}/txstore_batch.go`
- `wallet/internal/db/{sqlite,pg}/txstore_applyscanbatch.go`
- `wallet/internal/db/kvdb/txstore.go`
- `wallet/internal/db/txstore_common.go`
- `wallet/internal/sql/sqlite/queries/wallets.sql`
- `wallet/internal/sql/pg/queries/wallets.sql`
- watch-generation and exact raw-script/outpoint SQL queries
- `wallet/internal/sql/{sqlite,pg}/migrations/000014_canonical_chain.*.sql`
- `docs/developer/adr/0001-multi-wallet-architecture.md`
- `docs/developer/adr/0006-wtxmgr-sql-schema.md`
- `docs/developer/adr/0003-optimistic-cfilter-batching.md`
- `docs/developer/adr/0004-targeted-rescan-vs-rewind.md`

#### Tests

- Delay or suppress standalone relevant-transaction delivery, wait only for the
  wallet tip, and assert transaction confirmation and balance immediately.
- Deliver the same relevant transaction after the atomic block and prove no
  duplicate credits or notifications.
- Mempool `RelevantTx` immediately before an unrelated block remains unmined;
  later confirmation applies only in its actual block.
- Confirmed hints from different epoch/hash buffers never mix.
- Bare-multisig/raw-script receives and watched-prevout spends are found by the
  backend-specific authoritative proof.
- Mutation during account/address pagination or between horizon, address, and
  UTXO reads cannot produce a mixed snapshot. A generation mismatch before
  commit forces a rebuild.
- Different recovery-lookahead policies use different runtime epochs/digests.
- Neutrino non-match fetches no full block; match, false-positive, raw script,
  unavailable-filter, and generation-change vectors preserve completeness.
- Targeted scan adds old history/horizons without changing the exact global tip.
- Targeted wrong-hash, missing, pruned, epoch-mismatch, and rollback-race cases
  never create a canonical block row or partially ingest history.
- Canonical stale replay requires durable complete-batch proof.
- A height jump or wrong previous hash enters catch-up and cannot advance the
  visible tip.
- Same-height reorg during disconnect is found and reconciled without a later
  block on btcd, bitcoind, and neutrino.
- Reorg lowers the tip only through rollback and reconfirms correctly.
- Two wallets/processes with conflicting same-height backends enter one
  database arbitration. Disagreement preserves the old frontier and unsynced
  state; convergence or arbiter takeover performs one epoch transition.
- Wallet B attaches its own effects/proof to wallet A's existing canonical row
  across lag, restart, concurrent/same-wallet races, and generic-intent GC.
- Arbitration tests stale/crashed/paused participants, dissent, convergence,
  late registration, and takeover across processes. Paused ownership blocks
  adoption; fenced crash removal and unanimous adoption occur exactly once.
- Graceful Stop during arbitration remains in the frozen set until independent
  abandoned-lock fencing; a prior NACK remains dissent. Outside arbitration it
  retires normally.
- NACK-then-fence cannot adopt, and fencing every participant cannot satisfy
  unanimity; retained generation rows preserve both outcomes across takeover.
- Crash each phase prefix and prove only complete PREPARED-through-READY facts
  resume. PostgreSQL server-clock tests cover just-before/equal/after expiry;
  SQLite restart refreshes all proofs. Backward/forward host jumps, refresh/
  adoption races, TTL validation, and skew never permit early adoption.
- A populated migration-12/13 fixture with sparse rows, conflicting wallet tips,
  matching/mismatching confirmations, and interrupted cutover reaches one
  re-proven ready migration-14 state.
- Migration 14 performs no chain call; restricted post-open cutover rejects
  ordinary roles/empty participant sets, and coordinator crash resumes the same
  generation through ready before starting wallets.
- Pre-cutover quarantine admits maintenance while no cutover owner exists.
  Ambiguous cutover failure releases/fences ownership; verified repair resumes
  the same durable generation and reaches ready without exposing ordinary roles.
- Fresh SQLite/PostgreSQL migration 14 initializes ready bound genesis without
  participants, then Manager Create/Start succeeds normally.
- A targeted scan through ceiling H ignores queued H+1 until completion; start
  H+1 writes nothing, and rollback/epoch changes return/restart typed state.
- Public targeted-scan tests assert request/result ceiling, stale generation,
  every typed disposition, deadline/cancellation, partial-batch rollback, and no
  goroutine/request growth for Manager, recovery, and startup callers.
- Targeted/ordinary discoveries atomically queue registration; crash between
  batch commit and dispatch, absent owner, stale batch, retry, and owner
  replacement prove no discovered target is temporarily reported ready while
  unwatched.
- Blocked block/filter/header/best-tip requests cancel on reset/shutdown with no
  pending-request or worker growth. Arbitration remains typed and restartable.
- Recovery derivation/encryption blocks while no write transaction or write
  gate is held. A concurrent allocation makes the complete stale scan batch
  reprepare without transaction, balance, horizon, or tip effects.
- Live btcd, bitcoind, and neutrino event-ordering vectors.
- Legacy and deprecated consumers continue receiving and processing their
  existing bare and filtered notification stream.
- SQLite chain vector and focused PostgreSQL Store vector.

#### Exit Gate

- A visible synced tip includes all wallet effects through that block.
- Batch replay cannot regress a newer tip.
- Every canonical tip commit proves one current complete watch generation.
- Targeted rescans preserve ADR 0004 and never change the global watermark.

## BW4: Exact Accounts And Public Account Identity

### BWR-006: Custom Scope And Exact Account Creation

#### Issue And Rationale

`Wallet.NewAccount` accepts a scope and name but only allocates the next
account.
An unknown custom scope fails because creation supplies no schema and several
read validators consult the static `db.ScopeAddrMap` instead of persisted
`key_scopes`.

`ImportAccount` is not a substitute for a remote-signer family. It creates an
ordinary imported XPub account with no account number. lnd family identity is
an exact path such as `m/1017'/coin'/family'`, and the exact account component
must remain durable across restart.

ADR 0013 currently couples external/imported XPub provenance to a nil account
number. A path-bound external XPub proves an account child number but is still
not derived from the local wallet root. Treating it as locally derived would
make provenance false; dropping its number would make the path false.

#### Required ADR Decision

Update ADR 0013 before implementation so account provenance and known BIP32
path identity are orthogonal:

- A root-derived account has `is_derived=true` and an account number.
- An arbitrary imported XPub keeps `is_derived=false` and no account number.
- A path-bound external XPub keeps `is_derived=false` but may have a verified
  account number.
- `account_number` means the known account component for lookup; it no longer
  implies local private derivation by itself.
- Scope plus account number is unique whenever the number is present.
- ADR 0012 still rejects an external public-only account in a spendable SQL
  wallet.
- Preserve legacy KV `ImportAccount` as an explicit compatibility carve-out: on
  a spendable KV wallet it keeps the current imported-XPub semantics, produces
  an arbitrary external account with nil account number/watch-chain behavior,
  and remains outside the modern exact-account role. It cannot claim path-bound
  provenance or key-only mode. Do not route it through the SQL-only ADR 0012
  rejection; modern SQL/public requests still reject the impossible shape.

Do not implement the feature by silently setting `is_derived=true` on an
externally supplied XPub.

#### Public API Shape

Use one generic account request whose exact Go names are selected during S0:

```go
type NewAccountParams struct {
	Name                 string
	Scope                waddrmgr.KeyScope
	AddressSchema        *waddrmgr.ScopeAddrSchema
	AccountNumber        *uint32
	AccountPubKey        *hdkeychain.ExtendedKey
	MasterKeyFingerprint *uint32
	DiscoveryMode        DiscoveryMode
}
```

The fingerprint is optional by presence, not by comparing it with zero. Zero is
a valid 32-bit fingerprint. Wallet-owned `AccountInfo` and address derivation
results must preserve the same presence distinction after restart.

The zero discovery value is `watch-chain`. Account/Manager normalization and
manifest v2 carry the field now, but every non-default value returns unsupported
before mutation until BW5 lands the full key-only invariant atomically.

#### Mode Semantics

| Wallet | XPub | Number | Result |
|---|---|---|---|
| Spendable | nil | nil | Sequential root-derived account |
| Spendable | nil | set | Exact root-derived account |
| Spendable | set | either | Reject public-only material |
| Watch-only | set | nil | Arbitrary external XPub without number identity |
| Watch-only | set | set | Verified path-bound external XPub |
| Watch-only | nil | either | Reject hardened derivation without private root |

Additional request rules:

1. Require a public extended key on the configured network.
2. For a path-bound request, require depth three, a hardened child, and child
   number equal to the requested exact account. XPub plus nil number remains an
   arbitrary external account even if the key encodes a depth-three child.
3. Treat the requested purpose/coin scope as authoritative because an account
   XPub cannot prove its ancestors.
4. A nil schema may reuse an existing scope or a standard default. Creating a
   new custom scope requires an explicit schema.
5. Root-derived accounts always persist the computed fingerprint as present. A
   supplied fingerprint is an assertion and must match, including valid zero.
   External accounts preserve presence exactly as supplied.
6. Reject a schema conflict with an already persisted scope.
7. Replace Manager's parallel `WatchOnlyAccount` model with this request.
   `CreateWalletParams.InitialAccounts` becomes `[]NewAccountParams`; shell mode
   permits only the two external-XPub rows above. Route Manager replay and
   ordinary account creation through one validator and internal primitive.
8. PR 3 owns the complete internal create-manifest v2 wire/storage schema:
   presence-tagged number/fingerprint, authoritative scope, effective schema,
   XPub/provenance, discovery mode, name, wallet mode, and source tag. It
   records those reserved optional fields while accepting only legacy
   sequential/imported shapes. PR 6 first makes exact/path-bound persisted
   shapes legal, then maps the public request into the same schema and activates
   Manager capability validation in later commits. Neither changes the format.
   Decode v1
   incomplete-wallet manifests as arbitrary external watch-chain accounts with
   prior scope/schema and present old fingerprint; never reinterpret v1 as
   exact or key-only.
9. Phase A, before `beginCreate` or artifacts, resolves the backend and
   performs only state-independent shape/network/bounds/capability validation
   on the full request. KV rejects exact root-derived, path-bound external-XPub,
   and key-only initial accounts with stable unsupported; one bad mixed member
   rejects all.
10. `beginCreate` then persists the immutable request digest and acquires the
    create-owner/Store fence before any root-dependent work. Phase B loads or
    derives the wallet root under that guard, synthesizes spendable account zero
    named `default` in each implicit scope, merges requested accounts, and runs
    duplicate scope/number/name/schema normalization on the one sorted effective
    set. It freezes manifest v2 before the first account step. Guard loss rolls
    back prepared secrets and publishes nothing. Replay must match request and
    effective-plan digests. Exact zero remains valid in custom scopes;
    shell/watch-only creates synthesize no root defaults.

#### Store Semantics

1. Validate purpose and coin components before hardening and reject overflow.
   Validate accounts against public `waddrmgr.MaxAccountNum`; reject
   `ImportedAddrAccount`, the no-account sentinel, `MaxUint32`, and every
   reserved name/number before any Store mutation.
2. Represent exhausted sequential allocation explicitly. `MaxAccountNum+1` may
   be an internal next-counter sentinel but is never returned as an account.
   Compute `exact+1` only after bounds checks.
   `MaxAccountNum+1` is numerically `waddrmgr.ImportedAddrAccount`; as a scope
   counter it means exhausted and must never become an imported account row.
3. Extend account creation params with optional exact number, schema, XPub
   provenance, fingerprint presence, and zero-value discovery mode.
4. If account number is nil, allocate the next ordinary number.
5. If account number is explicit, claim it without filler accounts and advance
   the next counter to `max(current, exact+1)`.
6. Keep lower unclaimed holes available for later explicit creation.
7. Read wallet mode, scope/schema/generation, counter/exhaustion, and
   name/number indexes first. Outside the write, derive/neuter account keys,
   classify invalid
   children, and encrypt SQL or KV key material/imported XPub/fingerprint data.
   Commit scope, account, secrets, manifest link, intent, and final counter by
   CAS. A loser writes nothing, zeroes prepared secrets, and reprepares. Initial
   accounts use this same protocol; KV uses View/prepare/Update-CAS without
   derivation or encryption inside `walletdb.Update`.
   PR 3 lands the existing-shape sequential/imported primitive for initial
   accounts; PR 6 extends it only for exact/path-bound shapes and new metadata.
8. For sequential root derivation, commit every consumed invalid hardened BIP32
   account child in the operation payload and continue until a valid account or
   terminal exhaustion is reached.
9. For an explicitly requested account, an invalid child aborts the complete
   transaction and returns the typed public invalid-child error. It creates no
   scope, account, counter movement, manifest change, or mutation intent and
   never substitutes another number.
10. Every exact-account derivation failure and every sequential non-invalid
    derivation or insertion failure rolls back all account/scope-counter state.
    The next sequential request after an exact invalid child starts from the
    same counter value.
11. Compare a caller-supplied scope schema with the persisted scope schema.
   Return a typed conflict instead of silently accepting the first schema.
12. Remove static `ScopeAddrMap` rejection from account list query validation.
   Persisted scope existence is a backend fact.
13. Keep applied migration 5 immutable. Add forward migration
    `000015_accounts_v2` on both backends to update account shapes while
    preserving populated rows:
    - root-derived requires a non-null account number;
    - arbitrary external XPub may keep a null account number;
    - verified path-bound external XPub may carry a non-null account number.
14. Update every number lookup and account/address shape validator to accept
    exactly those three shapes. Do not let one backend accept a row another
    backend or read path reports as corruption.
15. Insert the generic SQL mutation intent atomically with account creation and
    link the account result to its operation ID.
16. Reconcile an ambiguous create by operation ID, then compare scope, number,
    name, XPub, provenance, schema, fingerprint, discovery mode, and manifest.
17. Return exhaustion as a successful transactional outcome, not a callback
    error. Commit consumed terminal state and the intent with nil callback
    error, reconcile COMMIT, then translate the outcome to a typed public
    exhaustion error. A request starting at an exhausted sentinel performs no
    mutation.
18. SQL may persist `MaxAccountNum+1` as the next-counter sentinel. KV persists
    the last consumed `MaxAccountNum` plus an explicit exhausted outcome; it
    never treats `ImportedAddrAccount` as an allocation counter.
19. Move imported-plus-derived handling into this cutover. Provenance and
    derivability are independent: create derivation metadata whenever durable
    path/account identity exists; never branch on `Imported` to decide whether
    account derivation is possible.
20. Audit `GetDerivationInfo`, PSBT decoration, and signer secret routing. Exact
    external children with number/fingerprint emit normal paths; arbitrary
    external children derive internally by `AccountID` but cannot emit a full
    public BIP32 path; raw imports alone use per-address secrets.
21. Preserve the existing raw-v0 KV `account-master-fingerprint` bucket. Add a
    separate v1 bucket keyed identically with version, presence, and optional
    four-byte value. Read v1 first; explicit absent is authoritative. Otherwise
    accept exact four-byte raw-v0 as present (including zero), then legacy
    imported numeric or root-derived fingerprint as present. New writes use v1;
    never rewrite/delete raw-v0 on read/open. Malformed/unknown entries fail
    closed without fallback.
22. Active-owner watch-chain creation prepares both branch horizons under the
    runtime recovery policy outside the write. Its CAS creates the account,
    increments watch generation, enqueues one exact-script horizon-registration
    event, and leaves the intent awaiting. Return `AccountInfo` only after typed
    registration and atomic cursor/disposition/readiness completion. Every
    non-registered outcome returns no successful result. Retryable, unknown,
    not-attempted, or stale dispatch leaves one repairable event; unsupported
    terminally disposes that delivery with readiness false; identity conflict
    quarantines and does not advance the cursor.
23. Expose account watch readiness in get/list results; a pending account XPub
    is not receiving authority. A successfully activated empty recovery-window-
    zero snapshot is ready but contains no receiving script; its first
    allocation is permitted and returns only after the exact new script
    registers.
    Create-owner accounts queue work for starting-owner snapshot subsumption/
    replay. Key-only creation is born complete, changes no watch generation, and
    enqueues nothing.
24. Freeze public `AccountWatchState` as `not_applicable`, `pending`, `ready`,
    or `unsupported`, returned by create/get/list on every backend with watch
    generation, stable `SubscriptionID`, and optional current `*DeliveryID`.
    Delivery is absent exactly for not-applicable/key-only and required for
    pending, ready, or unsupported watch-chain state. SQL persists it in
    migration 12's account-watch state table with the account and outbox
    disposition; PR 3 initializes legacy/default accounts, so PR 6 needs no
    separate watch-state DDL. KV atomically persists equivalent versioned side
    metadata in walletdb
    and replays it through the process-local starting/active owner.
    Watch-chain create begins pending; only verified registration/snapshot
    membership CASes ready. Typed unsupported is terminal-unsupported, while
    retryable/unknown remains pending. Receiving/address allocation rejects
    pending/unsupported; key-only is not-applicable. A create whose delivery
    fails returns no successful result, but get/list exposes the durable state
    for retry/diagnosis rather than making the account invisible.
    A later capable backend generation creates a new delivery transition from
    unsupported to pending, then ready only after membership proof.
25. Carry master fingerprint as `*uint32` through wallet-owned account/address
    derivation metadata. PSBT BIP32/Taproot decoration emits a derivation record
    only when the fingerprint is present; absent returns public
    `ErrDerivationPathNotFound` before packet mutation rather than fabricating
    zero. Present-zero emits literal fingerprint zero. Signing validates packet
    fingerprint presence/value against the same persisted identity.

#### Primary Files

- `docs/developer/adr/0013-normalized-account-address-identity.md`
- `wallet/account_manager.go`
- `wallet/account_derive.go`
- `wallet/manager.go`
- `wallet/signer.go`
- `wallet/psbt_manager.go`
- `wallet/internal/db/data_types.go`
- `wallet/internal/db/accountstore_createderived.go`
- `wallet/internal/db/accountstore_accountinfo.go`
- `wallet/internal/db/accountstore_getaccount.go`
- `wallet/internal/db/addressstore_common.go`
- `wallet/internal/db/keyscopestore_ensurekeyscope.go`
- `wallet/internal/db/accountstore_listaccounts.go`
- `wallet/internal/db/{sqlite,pg}/accountstore_createderived.go`
- `wallet/internal/db/kvdb/accountstore_createderived.go` for explicit
  unsupported exact/path-bound modes while preserving sequential behavior
- `wallet/internal/db/kvdb/accountstore_createimported.go`
- `waddrmgr/scoped_manager_accounts.go`
- `wallet/internal/sql/{sqlite,pg}/queries/key_scopes.sql`
- `wallet/internal/sql/{sqlite,pg}/queries/accounts.sql`
- `wallet/internal/sql/{sqlite,pg}/migrations/000015_accounts_v2.*.sql`

#### Tests

- Exact account 0, sparse 9999, and an explicit lower hole after 9999.
- Maximum valid account, first invalid value, `ImportedAddrAccount`, no-account,
  and `MaxUint32`.
- Maximum valid and overflowing purpose/coin components.
- Sequential then exact, exact then sequential, and same-number races.
- Unknown custom scope creation and subsequent get/list.
- Same schema idempotence and different schema conflict.
- Correct root-derived and external-XPub provenance.
- Valid root-derived, arbitrary external, and path-bound external shapes pass
  every create/get/list/address conversion; all impossible shapes fail.
- Spendable/watch-only mode matrix.
- XPub depth, child, network, private/public, and account mismatch failures.
- Fingerprint zero with presence and absent fingerprint as distinct cases.
- PSBT input/output/change and Taproot decoration preserve present-zero, while
  absent fingerprint returns `ErrDerivationPathNotFound` without packet mutation
  across restart.
- Every XPub/number/wallet-mode combination in the request matrix.
- Root fingerprint match/mismatch and authoritative custom scope despite XPub
  version bytes.
- Manifest v2 mismatch for every identity field and canonical v1 partial replay.
- Request-local duplicates fail before `beginCreate`. Implicit-default/root
  collisions fail only under the create guard, before account/secret artifacts;
  guard cleanup is deterministic. Custom-scope exact zero succeeds; caller order
  yields one manifest and every committed-prefix retry uses it.
- Rollback and both ambiguous-commit outcomes.
- Injected sequential invalid-account child and account-range exhaustion.
- Exact invalid-account child fails without choosing another account.
- Existing/new-scope exact invalid children leave no scope, account, counter,
  manifest, or intent state; the next sequential call uses the prior index after
  retry and restart.
- Restart identity on SQLite and PostgreSQL; KV rejects exact/path-bound modes
  without changing existing sequential account behavior.
- KV initial exact, path-bound, key-only, and mixed account lists fail before
  creating a legacy DB, runtime DB, manifest, cache entry, or account.
- KV arbitrary external fingerprints preserve absent, present-zero, and
  present-nonzero through create/get/list/derivation/restart. Legacy zero
  without metadata is present-zero; dry run writes nothing; unknown metadata
  fails.
- Legacy KV `ImportAccount` in a spendable wallet preserves frozen success,
  restart, derivation, and watch behavior with nil account number; equivalent
  modern SQL/public creation rejects and cannot acquire exact/key-only metadata.
- Imported-derived `GetDerivationInfo`, PSBT decoration, and account-secret
  routing; raw imports retain address-secret routing.
- SQL/KV sequential invalid-child then success, committed exhaustion, restart
  still exhausted, and non-invalid derivation rollback.
- Block derivation/encryption and prove no write transaction/gate is held.
  Same-counter CAS races leave no discarded ciphertext, intent, or cache state;
  ambiguous COMMIT reconciles the original prepared candidate.
- A frozen-tip KV fixture with raw zero/nonzero fingerprint bytes survives
  get/list/secret/derivation/rename/restart unchanged. Test v1 precedence,
  absent/present-zero/nonzero, malformed entries, rollback, and dry run.
- Active watch-chain creation waits for horizon registration. Retryable/unknown
  failure or crash returns no ready result and startup repairs one event;
  unsupported terminally disposes it, stale-owner replay hands off, and identity
  conflict quarantines. Mempool receipt after success, zero-window readiness,
  initial-default startup, and key-only controls pass on SQLite/PostgreSQL and
  all live backends in PR 8.
- SQL and KV create/get/list preserve every account watch-state transition
  across restart, retry, unsupported backend, owner replacement, and snapshot
  subsumption. Pending/unsupported cannot allocate a receiving address; key-only
  remains not-applicable.

#### Exit Gate

- Exact accounts require no fillers.
- External path-bound XPub accounts retain truthful provenance and exact path.
- Public account/address results distinguish absent fingerprint from zero.
- Custom scopes are readable after creation.
- Migration constraints, number-based queries, and Go validators agree on all
  account shapes.
- No account API returns an internal DB type.

## BW5: Atomic Bulk Addresses And Durable Discovery

### BWR-007: Bulk Allocation And Invalid Children

#### Issue And Rationale

`Store.NewDerivedAddress` atomically allocates one address. Repeating it in a
wallet loop allows caller batches to interleave. SQL derivation also runs before
the counter can commit an invalid BIP32 child. When derivation returns
`hdkeychain.ErrInvalidChild`, rollback leaves the same index current forever.

The allocator also checks `math.MaxUint32`, but public child derivation is
limited to the non-hardened range ending at `db.MaxAddressIndex`.

#### Detailed Fix

1. Add a Store `NewDerivedAddresses` operation and retain count-one as a wrapper
   if existing internal callers require it.
2. Select an account by exact persisted identity, not only inferred address
   type.
3. Define positive count as the number of valid addresses returned. Freeze
   wallet-owned `MaxBulkAddresses uint32 = 1_000` and
   `ErrInvalidAddressCount`. Count zero or above the maximum returns that public
   error before operation-ID generation, counter reads, transactions, intents,
   or chain registration.
4. Enforce the same numeric maximum in both direct waddrmgr batch methods with a
   stable manager error that the KV wallet boundary translates to
   `ErrInvalidAddressCount`. SQLite, PostgreSQL, KV, and direct waddrmgr cannot
   select different limits.
5. Read one account/branch snapshot containing stable account identity, XPub,
   schema, discovery mode, branch counter, and state generation.
6. Outside write admission, prepare the complete valid-count batch in increasing
   index order, including invalid indexes and final counter. Legacy KV performs
   any private-key encryption during preparation; SQL public derivation invents
   no ciphertext.
7. Reject indexes above `db.MaxAddressIndex`; permit only the persisted next
   counter to hold the exhausted sentinel `MaxAddressIndex+1`.
8. Generate a cryptographically random allocation operation ID before the CAS
   write.
9. CAS the expected snapshot, then atomically insert every prepared row/path,
   final counter, intent, and watched outbox payload. Associate every result
   with the operation ID.
10. A CAS loser creates no durable/cache effect and reprepares from a fresh
    snapshot. Publish waddrmgr/runtime caches only after commit.
11. Return results in candidate-index order. Another caller's batch must not
   interleave on the same account and branch.
12. Abort the complete batch on any non-invalid-child derivation or insert
    error.
13. Represent terminal exhaustion as a successful transactional outcome. Commit
    the final counter, consumed range, exhaustion intent, and zero address rows
    with nil callback error; after definite/reconciled COMMIT, translate it to a
    typed public exhaustion error. Valid candidates encountered before terminal
    exhaustion become consumed gaps and are never reused.
14. A request starting at the exhausted sentinel returns the typed error with no
    mutation, intent, or registration. Exhaustion never produces a chain side
    effect.
15. On an ambiguous commit, reconcile by operation ID first, then compare the
    complete candidate set by account, branch, index, script, public key, and
    address type. A later caller that allocates the same deterministic locator
    has a different intent and cannot be mistaken for this caller.
16. Matching intent and complete candidates means committed. Subject to the
    Store-session fencing rule, no intent plus matching old state means rollback
    and permits a fresh range. Otherwise the outcome is unknown. A partial or
    mismatched result quarantines the wallet.
17. A key-only allocation inserts its intent as `complete` in the allocation
    transaction and performs no completion CAS or second write. Ambiguous
    COMMIT still reconciles the complete intent and exact rows. Watched
    allocation inserts `applied`, moves to `awaiting_side_effect` after proof,
    performs idempotent registration, then atomically advances the runtime
    cursor and CASes the intent to `complete`.

#### Public API Semantics

Expose `NewBulkAddresses` through `AddressManager` with:

- The public `(Scope, AccountID)` `AccountSelector` returned by `AccountInfo`.
- External or internal branch.
- Valid-result count.
- Public `MaxBulkAddresses` and `ErrInvalidAddressCount`.
- Rich `AddressInfo` containing public key and committed derivation locator.

Preserve the established address contracts:

- `NewAddress` continues to force one new watched allocation.
- `GetUnusedAddress` continues to return the oldest unused receiving address
  and allocates only when none exists.
- Every successful positive-count `NewBulkAddresses` call advances the branch.

Do not infer a custom account only from an address type.
Update both authoritative lnd planning documents to remove the proposed
`NewAddress`/`GetUnusedAddress` rename and map lnd callers to these preserved
contracts.

#### Primary Files

- `wallet/address_manager.go`
- `wallet/internal/db/interface.go`
- `wallet/internal/db/data_types.go`
- `wallet/internal/db/addressstore_newderived.go`
- `wallet/internal/db/{sqlite,pg}/addressstore_newderived.go`
- `wallet/internal/db/kvdb/addressstore.go`
- SQL account/address migrations and mutation-intent queries
- `waddrmgr/scoped_manager.go`

#### Tests

- Count zero, one, exact `MaxBulkAddresses`, max-plus-one, external, and
  internal branches on SQLite, PostgreSQL, KV, and both direct waddrmgr methods.
- Invalid count performs no operation-ID generation, read, mutation, intent,
  callback indexing, or notification at every entry point.
- Injected invalid-child skipping and persisted gaps, plus a normal real-XPub
  derivation vector.
- Last non-hardened index and exhaustion marker.
- Terminal exhaustion after prior valid candidates commits zero address rows,
  persists consumed gaps, returns the typed error, and remains exhausted after
  restart.
- Same-branch concurrent batches never overlap or interleave.
- Different accounts and branches remain independent.
- Rollback after several candidate addresses leaves no intent, rows, or counter
  movement.
- Block derivation/encryption and prove no write transaction/gate is held. A
  same-counter loser writes nothing and reprepares on SQL/KV; discarded random
  ciphertext never appears in rows, intents, or caches.
- Ambiguous committed and ambiguous rolled-back outcomes.
- Caller A rolls back ambiguously, caller B commits the same deterministic next
  locator, and caller A does not claim caller B's intent.
- Stop/load continues at the next durable index.
- SQLite and PostgreSQL integration plus KV/waddrmgr regression.
- SQL and modern KV parity for positive bounds, ordered count-N, invalid-child
  consumption, zero-row terminal exhaustion, and rollback of non-invalid
  derivation errors. Exact accounts, key-only mode, intents, and quarantine
  remain SQL-only.
- A maintained count-one and count-N allocation benchmark measures mutation
  intent and completion overhead, wall time, allocations, and SQL writes.

#### Exit Gate

- One successful call returns exactly the requested valid count.
- Returned locators are ordered and never reused.
- Invalid children cannot wedge an account branch.
- Concurrent count-one calls return distinct committed locators.
- The lnd plan and gap register preserve the established `NewAddress` and
  `GetUnusedAddress` contracts.

### BWR-008: Durable Key-Only Intent And Registration Repair

#### Issue And Rationale

The older plan proposed a request-only `SkipChainWatch` flag. That suppresses
the immediate `NotifyReceived` call but is not durable. On restart, syncer
reconstructs watch targets and recovery horizons from persisted accounts and
addresses. A purpose-1017 key can therefore enter the chain watch set later.

Receiving-address registration has the opposite failure. `Wallet.NewAddress`
commits an address and then calls `NotifyReceived`. If that call fails,
`GetUnusedAddress` may later return the same committed address without retrying
registration.

Key-only must also have an ownership meaning. Merely excluding an address from
initial discovery is insufficient if transaction ingestion later creates a UTXO
for it and `ListOutputsToWatch` reintroduces it after restart.

Commit-before-register is the correct order. Register-before-commit can leave
the chain backend watching a locator that another writer wins or that the wallet
never stores.

**Open question carried in from Task 315 rev 6.** That stack briefly added
startup seeding of the backend watch list with the wallet's *existing* addresses
and unspent outputs, then removed it because §5.4 of 315 defers the live-watch
redesign. Its rationale is worth keeping in front of whoever schedules this
item: once the wallet is synced the backend filters live blocks against its own
watch list, so an unseeded list means a live block paying an existing address is
filtered out as irrelevant while the sync tip advances past it — a permanently
missed deposit, and the same for a spend of an existing UTXO. Base does not seed
the list either, so this is a pre-existing gap rather than a regression, and the
Manager itests pass without it. If the gap is real, the startup-registration
backstop below is where it gets closed. See `opus-report.md` §11.2 F3.

#### Detailed Fix

1. Persist an account-level discovery mode with a safe default of
   `watch-chain` and an explicit `key-only` value.
2. Activate the request's `key-only` value only in this change. Earlier account
   work carries the zero-value `watch-chain` field for manifest stability but
   rejects every non-default mode before mutation.
3. Use account-level policy because recovery lookahead derives future children.
   A per-call or per-address flag cannot prevent future account discovery.
4. Disallow mixing watched and key-only allocations within one account in this
   milestone. Changing an account's mode after creation is unsupported.
5. Define key-only accounts as signing/derivation locators, not wallet receiving
   ownership. Exclude their scripts from `ResolveOwnedAddresses`, credit/UTXO
   creation, balances, `ListOutputsToWatch`, startup registration, recovery
   lookahead, compact-filter targets, and full-block watch material.
6. Enforce the exclusion inside Store transaction/credit insertion as well as
   wallet-level watch construction. Wallet callers must not be able to bypass
   it by manually ingesting a transaction.
7. Keep key-only locators available through address/key lookup and signer roles.
8. For a key-only account, reject `NewAddress`, `GetUnusedAddress`, receiving
   imports, and funding change selection.
9. Let `NewBulkAddresses` derive registration behavior from the durable account
   policy. Do not ship a transient `SkipChainWatch` contract as the source of
   truth.
10. For watched accounts, atomically increment a per-wallet outbox counter and
    store `(wallet, sequence, operation ID, SubscriptionID, DeliveryID, origin
    runtime epoch, kind, watch generation, mode, payload digest, complete
    ordered count-N payload)` with the allocation.
    Transactional sequence reservation rolls back without gaps. Store-session
    sequence is never delivery order.
11. Reconcile COMMIT, transition the intent to `awaiting_side_effect`, and
    deliver the logical payload through PR 2's modern registration API using
    bounded staged chunks and the current dispatch epoch.
    The call succeeds only after one transaction contiguously advances the
    cursor, records event disposition, and marks the intent `complete`.
12. A crash after backend activation but before that delivery transaction may
    invoke registration again. Require at-least-once idempotent invocation, one
    observable watched state, and one durable disposition rather than exactly
    one backend call.
13. If registration is retryable, unknown, or not-attempted, return an error and
    no usable success result. Keep one repairable awaiting event; do not
    allocate another batch. Before `GetUnusedAddress` returns a member, replay
    the same DeliveryID/payload and finish delivery. Unsupported records
    terminal-unsupported, advances the cursor/completes the intent, keeps
    account readiness false, and returns the stable capability error.
    Stale returns no result, fences the caller, and keeps the event pending for
    successor replay or proven snapshot subsumption. Conflict quarantines the
    registration invariant and does not advance the cursor.
14. A durable `starting-owner` capability obtains the authoritative exact-script
    snapshot, watch generation/digest, policy identity, and outbox frontier in
    one consistent Store read. After snapshot registration, pre-frontier
    registration events may be `subsumed_by_snapshot` only after membership
    proof; other kinds require delivery or explicit disposition.
15. Replay post-frontier events strictly by wallet sequence. Retryable, unknown,
    not-attempted, and unresolved stale events block later events; conflict
    quarantines and blocks, while terminal unsupported is a disposition, not a
    gap. Delivery transactions accept matching `starting` or `active` owner;
    public roles accept only `active`. A replacement owner replays the old
    DeliveryID with its own dispatch epoch CAS and cannot duplicate activation.
16. The snapshot includes both branches of every `watch-chain` account in
    standard or custom scopes and excludes all key-only accounts. Mark the
    runtime active only after birthday repair, bounded lease cleanup, snapshot,
    and contiguous replay complete and a final CAS confirms health, generation,
    digest, frontier, and cursor. Expose roles/workers only after that CAS;
    cancellation fences the starting epoch.
17. Keep legacy address-type-only APIs standard-scope-only because an address
    type cannot select one custom scope/account. Custom scopes use exact account
    selectors through `NewBulkAddresses`.
18. Update `lnd-native-wallet-plan.md` and
    `wallet-default-sqlite-lnd-gaps.md` to replace request-only
    `SkipChainWatch` with the durable account discovery contract before the
    wallet API is considered frozen.
19. Add forward migration `000016_discovery_allocation` for discovery mode,
    prepared allocation identity, and operation-specific registration payload.
    Generic runtime cursors already belong to migration 12.

#### Primary Files

- `wallet/address_manager.go`
- `wallet/syncer.go`
- `wallet/recovery.go`
- `wallet/internal/db/data_types.go`
- runtime-epoch, outbox event, payload, and delivery-cursor SQL queries
- account migrations and account read/write queries for both SQL backends
- `wallet/internal/sql/{sqlite,pg}/migrations/`
- account/address ownership resolution and transaction credit insertion for
  SQLite and PostgreSQL
- output-to-watch and UTXO/balance queries affected by discovery mode

#### Tests

- Key-only bulk allocation makes no immediate chain notification.
- Restart and recovery never add key-only account children to watch filters.
- Manually ingesting a transaction paying a key-only script does not create a
  wallet credit, balance, or output-to-watch.
- Funding rejects a key-only account as a change source.
- Watched allocation registers only after definite/reconciled commit.
- Two independent Managers cannot activate the same wallet runtime; a fenced
  predecessor's complete count-N payload is delivered by its replacement.
- Activation racing with allocation preserves snapshot/frontier ordering.
- A starting owner can repair/register/replay but every public role rejects it.
  Cancellation or generation change before final CAS exposes no runtime;
  create, starting, active, cutover, maintenance, and fenced checks agree.
- Allocations immediately before and after the atomic snapshot/frontier read,
  cursor gaps, old-epoch workers, and runtime replacement preserve order.
- After retryable/unknown `NewAddress` registration failure,
  `GetUnusedAddress` retries the same DeliveryID/candidate. Unsupported remains
  non-receiving without retry loops, stale hands off/subsumes, and conflict
  quarantines; none returns the candidate prematurely.
- Startup repairs watched addresses left by a process interruption.
- Custom-scope external and internal watched addresses repair after restart;
  key-only accounts contribute neither branch.
- Ambiguous allocation does not notify until durable rows are proven.
- Imported-XPub child reports both imported provenance and path metadata.
- btcd, bitcoind, and neutrino registration vectors where the harness supports
  them.

#### Exit Gate

- Purpose-1017 key-only accounts never grow chain watch filters across restart.
- Key-only scripts cannot become wallet UTXOs through manual ingestion.
- No receiving address is returned before idempotent registration succeeds.
- Registration failure may create a harmless gap but cannot expose an unwatched
  receiving script.
- Both authoritative lnd planning documents require the durable discovery mode
  and contain no request-only `SkipChainWatch` contract.

## BW6: PSBT And Funding Correctness

### BWR-009: Metadata-Preserving Funding

#### Issue And Rationale

`populatePsbtPacket` rebuilds the unsigned transaction and replaces input and
output maps. Caller global data, derivations, proprietary fields, sequences,
version, locktime, and decorated input/output metadata can be lost. Matching
outputs by value and script is insufficient because duplicate outputs are
legal and must retain positional metadata.

Funding also changes transaction facts covered by existing signatures. Keeping
partial, final, or Taproot signatures while adding/reordering inputs or outputs
can return a packet whose signatures no longer authorize its unsigned
transaction.

#### Detailed Fix

1. Before Store/chain/vault/lease/signing access in every PSBT role, run one
   wallet-owned non-mutating structural validator. Require aligned dimensions,
   non-nil transaction elements, valid aggregate output value, and valid global,
   input, and output metadata. Reject nil pointer-valued entries, malformed or
   oversized unknown keys, duplicate serialized PSBT keys, invalid keys/xpubs,
   derivations, signatures, UTXOs, final witnesses, Taproot roots/trees/leaves/
   control blocks, and inconsistent encodings before dependency sort/serialize.
   Then run `VerifyInputOutputLen` and `Packet.SanityCheck` as supplemental
   checks, not comprehensive validation.
2. Preserve the existing at-least-one-input-or-output contract unless S0 records
   an intentional public behavior change. Normalize malformed packet failures
   under a wallet error while preserving the cause.
3. Reject any input carrying a standard signature or finalization field before
   funding, including ECDSA partial signatures, final scriptSig/witness,
   Taproot key-spend signatures, and Taproot script-spend signatures. Return a
   typed already-signed error rather than preserving invalid authorization.
   Also reject funding when any existing input requests base
   `SIGHASH_SINGLE`, including `ANYONECANPAY|SINGLE`, because adding/sorting
   inputs or outputs can retarget an in-range signature to another output.
   Return the stable unsafe-sighash error before side effects; keep the later
   signer/finalizer out-of-range check as defense in depth.
4. Build a wallet-owned safe deep clone without sorting/serializing the caller,
   optionally canonicalize/parse only the clone, and perform all mutation on it.
   Leave caller bytes and slice order unchanged on every result/error.
5. Preserve global XPubs, transaction version, locktime, caller input sequences,
   and only a closed wallet-owned allowlist of metadata proven transaction-
   independent. Funding rejects unclassified global/input/output unknown or
   proprietary fields before side effects because they may contain stale
   commitments, signatures, or nonce state. Combine/decorate may preserve them
   only when unsigned transaction facts remain byte-identical.
6. Associate existing input metadata by outpoint.
7. Associate existing output metadata by original position, not value/script.
8. Give every original output and newly added change output explicit internal
   provenance while randomization and BIP69 sorting run.
9. Sort transaction elements and their PSBT maps as paired records.
10. Decorate newly selected wallet inputs, then merge wallet fields into caller
   maps with conflict detection. Do not silently replace a conflicting
   derivation, script, UTXO, or proprietary value.
11. Return a wallet-owned PSBT merge-conflict error.
12. Determine the final change index from provenance rather than searching for
    the first equal output.
13. Freeze `CombinePsbt` semantics in the same merge engine. Every packet must
    have byte-identical unsigned transaction facts, including version, locktime,
    inputs/sequences, and ordered outputs. Singleton fields merge only if absent
    or equal. Keyed sets (XPubs, derivations, partial signatures, Taproot leaf
    signatures, unknown/proprietary keys) union by canonical serialized key;
    equal duplicates are idempotent and unequal values conflict. Different
    final scriptSig/witness conflict, and final fields mixed with any
    partial/signing metadata conflict rather than silently preferring one side.
    Merge all global, input, and output maps on a clone in deterministic key
    order; any conflict or malformed packet returns the typed merge-conflict and
    mutates no argument.

#### Primary Files

- `wallet/psbt_manager.go`
- `wallet/psbt_manager_test.go`
- any shared PSBT sort/merge helper introduced by the change

#### Tests

- Global XPubs and explicitly classified transaction-independent metadata
  survive funding. Unclassified global/input/output unknown and proprietary
  fields fail before side effects without packet mutation.
- Combine/decorate preserve opaque fields when unsigned transaction facts remain
  identical.
- Custom sequence, transaction version, and locktime survive.
- Duplicate equal outputs preserve their original positional metadata.
- Randomized change and BIP69 sorting return the exact change index.
- Caller/wallet decoration conflicts fail without mutating the input packet.
- ECDSA partial, finalized, Taproot key-spend, and Taproot script-spend packets
  fail before funding and remain byte-for-byte unchanged.
- In-range/out-of-range `SIGHASH_SINGLE` and `ANYONECANPAY|SINGLE` funding fail
  before mutation, sorting, Store, or chain access; non-SINGLE controls retain
  their logical input/output association.
- Nil unsigned transaction, too few/many input or output maps, raw scriptSig,
  and raw witness fail without Store, allocation, or chain calls.
- Nil `TxIn`, nil `TxOut`, aggregate value above `MaxSatoshi`, and aggregate
  integer overflow do not panic or mutate the packet and perform no Store,
  allocation, lease, or chain call.
- In-memory malformed and duplicate global/input/output metadata cannot panic or
  mutate caller ordering and performs no Store/chain/vault/lease call through
  funding, signing, finalization, decoration, or combining.
- Combine tests cover identical replay, complementary global/input/output maps,
  unequal singleton/keyed/proprietary values, unsigned-tx/sequence/output-order
  mismatch, final-versus-partial conflicts, deterministic argument-order output,
  and byte-for-byte immutability on every error.

#### Exit Gate

- Funding preserves every admitted caller field it does not explicitly own;
  unclassified opaque metadata is rejected, never silently dropped or carried.
- Metadata remains attached to the same logical input or output after sorting.

### BWR-010: Complete PSBT Signing Authorization

#### Issue And Rationale

`parseBip32Path` compares a derivation path with the network's conventional coin
type. lnd intentionally uses persisted coin-type-zero scopes for standard
families on some test networks. The wallet already resolves the actual
scope/account from durable identity, so a network-global comparison rejects a
wallet-owned key without adding security.

**Reproduced 2026-08-01 (Task 315 rev 6), on both real backends.** The defect is
sharper than "rejects an lnd scope": the wallet cannot sign a PSBT it described
itself. `GetDerivationInfo` emits the key scope's coin type
(`wallet/address_manager.go:1111`, 0 for BIP-0084) while `parseBip32Path`
requires `ChainParams.HDCoinType` (`wallet/psbt_manager.go:1041`, 1 on test
networks), so:

| path built from | result |
| --- | --- |
| the wallet's own `GetDerivationInfo` | error: `expected coin type 1, got 0` |
| the network coin type instead | no persisted scope resolves; the input is skipped **unsigned** |

The second failure is silent — `SignPsbt` returns no error and an empty
`SignedInputs`. This blocks address-driven PSBT coverage on every non-mainnet
network, and is why Task 315 could not deliver that half of its DoD-6 matrix
(see `opus-report.md` §11.2 F1). Item 2 below is the fix.

#### Detailed Fix

1. Make path parsing validate BIP32 shape, hardened components, branch, and
   child bounds only.
2. Resolve purpose, coin, and account against the wallet's persisted scope and
   account identity.
3. Always derive the candidate key through the selected account and verify that
   it matches the packet's compressed or x-only declared public key before the
   wallet signs.
4. Derivation-authorized wallet signing requires persisted fingerprint presence.
   Absent returns public `ErrDerivationPathNotFound` before key selection or
   packet mutation. When present, compare exact identity, including literal
   zero. Keep lnd-specific remote-signer and RPC policy in the lnd shim; do not
   add lnd-specific coin rules to btcwallet.
5. Never replace persisted identity with `chainParams.HDCoinType` during
   signing.
6. Validate the complete authorization chain before signing:
   - a non-witness UTXO transaction hash matches the input outpoint and the
     output index exists;
   - witness and non-witness UTXO data agree when both are present;
   - P2SH redeem scripts and P2WSH witness scripts commit to the prevout
     program;
   - the derived key commits to the P2PKH, P2WPKH, nested, or multisig script
     position being signed;
   - Taproot internal key, merkle root, output key, leaf, and control block
     agree with the prevout;
   - the requested sighash type is supported for that spend path.
7. Treat caller-provided UTXO and script data as untrusted. Any failed check
   skips or rejects signing according to the existing role contract and emits
   no signature.
8. Route every wallet-generated PSBT signature through one internal authorized
   signing path shared by `SignPsbt` and `FinalizePsbt`. Finalize may complete
   valid third-party signatures, but any input it auto-signs gets the same
   parent, script, path, key, and sighash validation.
9. Require a matching full parent for legacy inputs. Permit witness-only
   P2WPKH, nested P2WPKH, and P2WSH/SegWit-v0 when amount, outer prevout,
   redeem/witness scripts, key position, and sighash are fully authorized. When
   both UTXO forms exist, require matching parent txid/index and exact output.
   Validate an optional SegWit parent rather than ignoring it. Derive spend type
   from scripts, never a static BIP-purpose map.
10. Inspect all BIP32/Taproot derivations and deterministically select at most
    one wallet-owned unsigned key in canonical `(base key, path, spend kind,
    leaf hash-or-empty)` order. Foreign candidates are skipped; malformed
    wallet-owned candidates fail. Preserve the typed single-leaf-per-derivation
    limit: reject multiple leaf hashes before selection or mutation. Multiple
    leaf-script records are selected by matching hash/control block, not order.
11. Define tweaked-key identity explicitly. Derivation metadata continues to
    identify the persisted base key/path, and `InputTweakers[input]` is applied
    exactly once before script authorization:
    - ECDSA records the effective compressed key in `PartialSig.PubKey`;
    - Taproot script-path records the effective x-only key in
      `TaprootScriptSpendSig.XOnlyPubKey`;
    - Taproot key-path validates the effective internal key, applies the BIP341
      key tweak after any caller tweak, validates the output key, and writes
      only signature bytes to `TaprootKeySpendSig`.
    `FinalizePsbt(ctx, packet)` has no tweaker argument and therefore performs
    only untweaked auto-signing. Callers requiring a tweak call `SignPsbt`
    first; finalize may complete a valid already-tweaked signature.
12. Refactor signer helpers so authorization can pass an already-derived
    effective key. Do not call a lower signer that applies the same tweak again.
13. Update `docs/developer/psbt_workflows.md`; callers no longer replace
    derivation metadata between signing passes.
14. In the shared authorization path, reject legacy `SIGHASH_SINGLE`, including
    `ANYONECANPAY|SINGLE`, when the input index has no corresponding output.
    Reject before key lookup/derivation/signing in both `SignPsbt` and
    `FinalizePsbt`, leave the packet unchanged, and return a stable unsafe
    sighash/malformed error.
15. Normalize effective sighash from validated spend version before
    authorization/signing. Absent means `SIGHASH_ALL` for legacy/SegWit v0 and
    `SIGHASH_DEFAULT` for Taproot. Preserve field absence but pass the effective
    value through helpers: ECDSA appends ALL, default Taproot remains 64 bytes.
16. `FinalizePsbt` works only on BWR-009's clone. Build one aligned validated
    prevout vector, verify parent/index and witness/non-witness equality plus
    all script commitments, then auto-sign and call `MaybeFinalizeAll` on clone.
17. Extract the finalized clone, build complete `MultiPrevOutFetcher` and
    `TxSigHashes`, then run `txscript.NewEngine` with standard flags for every
    input, including third-party partial and already-finalized inputs. Return a
    stable final-script-verification error on construction/execution failure.
18. Publish `*packet = *candidate` only after every input executes. Any
    structure, authorization, signing, finalization, extraction, or script
    failure leaves the caller packet byte-for-byte/order unchanged.
19. Run one side-effect-free structural/monetary validator before funding,
    decoration, combine, signing, or finalization. Require exact map counts,
    non-nil inputs/outputs, no duplicate/null prevouts, valid transaction
    version/locktime/sequence encoding, empty unsigned-tx scriptSig/witness,
    output and UTXO amounts in `[0, MaxSatoshi]`, checked aggregates,
    parent/index consistency, and no duplicate/reserved-key conflicts. Validate
    existing ECDSA/Taproot signature lengths and encoded sighash against
    declared/effective sighash; finalized fields must obey PSBT mutual-exclusion
    rules. Failure
    occurs before Store/vault/lease/chain access and never mutates the packet.

#### Primary Files

- `wallet/psbt_manager.go`
- `wallet/signer.go`
- `wallet/psbt_manager_test.go`
- `docs/developer/psbt_workflows.md`

#### Exit Gate

- A persisted wallet scope signs regardless of whether its coin component is
  the network convention.
- A path that does not resolve to the persisted account or declared public key
  does not receive a signature.
- Absent persisted fingerprint produces no decoration/signature and returns
  `ErrDerivationPathNotFound`; present-zero decorates and signs as literal zero.
- Forged non-witness transactions, mismatched witness UTXOs, redeem/witness
  scripts, Taproot output keys, leaves, and control blocks receive no signature.
- Witness-only lnd-style P2WPKH, nested, P2WSH, and purpose-1017 inputs sign;
  legacy witness-only fails and optional parent mismatch fails.
- `FinalizePsbt` rejects the same forged authorization data as `SignPsbt` and
  still finalizes a valid externally signed packet.
- Purpose-1017 P2WSH multisig and multi-derivation Taproot select one owned key;
  all-foreign derivations are skipped and metadata order does not affect choice.
- Single and double representative tweakers run exactly once, verify under the
  effective key, and record that effective key without changing base derivation
  metadata.
- Multi-leaf derivations fail without packet mutation. Multiple leaf-script
  records select by hash/control block regardless of order; a later call may
  select another unsigned candidate.
- Full script-engine verification replaces non-empty-signature assertions.
- Legacy hash-one `SIGHASH_SINGLE` and `ANYONECANPAY|SINGLE` fail before vault
  access when the input index is outside the output set; covered controls sign.
- Omitted-sighash P2PKH/P2WPKH/nested/P2WSH signs ALL and executes; omitted
  Taproot key/script paths use DEFAULT without suffix. Explicit controls remain.
- Forged third-party partial/final scripts and a forged second input fail engine
  verification without caller mutation; valid legacy, SegWit, multisig, and
  Taproot external inputs all finalize and execute.
- Duplicate/null prevouts, negative/overflow UTXOs/outputs, count mismatch,
  non-empty unsigned scripts/witnesses, reserved-key collisions, and existing
  signature/sighash inconsistency fail identically in fund/combine/sign/finalize
  before side effects and without packet mutation.
- Inject finalizer, extract, engine-construction, and engine-execution failures;
  none publishes the clone.
- Mainnet, testnet, simnet, and signet vectors pass.

### BWR-011: Change Registration Before Return

#### Issue And Rationale

`createChangeSource` calls `store.NewDerivedAddress` directly. The resulting
change script can be placed in a funded transaction or PSBT without calling
`Chain.NotifyReceived`. If the caller publishes externally, the active chain
backend may not watch the committed change until restart.

#### Detailed Fix

1. Route change allocation through the watched address allocation primitive.
2. Reject a key-only account as a change source.
3. Keep exported v1 `ChangeSource`, `InputSource`, and
   `NewUnsignedTransaction` source-compatible. Add `ChangeSourceV2` with a
   closed standard spendable template enum: P2PKH, P2SH, P2WPKH, P2WSH, or
   P2TR. Each owns exact length, script class, and dust-spend facts. Decide
   zero/dust before `NewScript`, then reject nil/empty, malformed, unspendable,
   null-data, P2A/anchor, unknown witness, non-standard, wrong-class/length, or
   actually dust returned scripts.
4. Land and tag the nested `wallet/txauthor` module first, then bump the root
   dependency in a separate buildable commit. Do not use a local `replace`.
   Update deprecated and sweep callers so the dependency bump preserves their
   dust behavior.
5. Keep address persistence inside Store and chain registration outside Store.
6. Do not return a funded transaction or PSBT until change registration
   succeeds.
7. On any non-registered outcome, return no funded result. The committed unused
   change address is a safe gap. Retryable/unknown/stale keeps its event for
   successor startup repair; unsupported terminally disposes that delivery and
   a later capable generation creates a new one; conflict quarantines.
8. On ambiguous allocation, reconcile the exact change locator before
   registration or return the public ambiguous error.
9. Scope the guarantee to modern `TxCreator.CreateTransaction` and
   `PsbtManager.FundPsbt`. `FundPsbtDeprecated` remains a walletdb compatibility
   API outside the modern outbox; the nested-module bump preserves its existing
   dust behavior without silently migrating it.
10. Validate checked aggregate output value in `TxCreator` before account,
    Store, input-source, chain, or change access. Defensively make
    `txauthor.NewUnsignedTransaction` reject nil outputs, integer overflow, and
    totals above `btcutil.MaxSatoshi` before `fetchInputs` or `NewScript`; use
    checked target-plus-fee arithmetic too.
11. Map every modern, deprecated, and sweep `waddrmgr.AddressType` to the exact
    closed template. Unknown types fail rather than guessing facts.
12. After each `fetchInputs` call and before fee/change work, require equal
    input/value/previous-script lengths, non-nil `TxIn`/scripts, valid input
    values, checked sum not above `MaxSatoshi`, and equality with reported
    total.
13. Update sweepaccount input source to hex-decode every non-zero prevout script
    and return it aligned with input/value. Malformed hex fails; never return a
    nil script vector. Audit all root adapters; mixed legacy/P2WPKH/nested/P2TR
    inputs contribute actual previous scripts to weight estimation.
14. Consume a corrected tagged txsizes estimator. It uses post-change output
    count CompactSize width and one witness-stack-count byte for every input
    when any input witnesses, including empty counts for legacy inputs.
15. Add `InputSourceV2` and `NewUnsignedTransactionV2`; do not change the v1
    function type/signature or structs used by external unkeyed literals. V2
    input results include an explicit conservative spend-weight descriptor, not
    script-class guessing. Standard descriptors cover P2PKH,
    P2WPKH, nested P2WPKH, and P2TR key spend. Generic P2SH/P2WSH/Taproot script
    paths provide bounded non-witness bytes and witness element counts/max
    lengths, including selector/script/control block. Validate descriptor and
    script compatibility plus CompactSize transitions; unknown or absent
    generic weight fails before change. `txsizes` consumes these descriptors and
    independently
    proves estimated weight is never below fully signed fixture weight.
16. Treat the nested module as a released artifact. Tag the txauthor module,
    verify the tag resolves to the reviewed commit and `go list -m -json`
    fetches that exact version without local replace/workspace leakage, then
    bump root `go.mod/go.sum`. CI enumerates every repository `go.mod`, runs
    tidy/test/build in each module context, tests root with workspace disabled,
    and starts/ends with no untracked benchmark binaries/profiles or modified
    module files.
    Add an external-module compile test proving existing v1 InputSource,
    ChangeSource, and NewUnsignedTransaction callers still build unchanged.
    PR 7 explicitly removes the root txsizes local replacement. With
    `GOWORK=off`, root effective `go list -m -json` for txsizes and txauthor
    must report the expected released version and no `Replace`; separate
    `path@version` queries prove tag provenance. The existing local wtxmgr
    replacement is explicitly out of scope and permitted.

#### Primary Files

- `wallet/tx_creator.go`
- `wallet/psbt_manager.go`
- `wallet/address_manager.go`
- `wallet/txauthor/author.go`
- `wallet/txauthor/author_test.go`
- `wallet/txauthor/go.mod`
- root `go.mod` and `go.sum`
- `wallet/deprecated.go` and `cmd/sweepaccount/main.go` callers

#### Exit Gate

- Every result returned by modern `TxCreator.CreateTransaction` or
  `PsbtManager.FundPsbt` has watched change.
- Chain registration never runs inside a Store transaction.
- Zero or dust change allocates and registers no address.
- Exact non-dust boundary allocates once; equal-size witness/non-witness
  templates make the correct different dust decision; returned template
  mismatch fails.
- Exact `MaxSatoshi` aggregate succeeds; `MaxSatoshi+1` and signed-overflow sets
  fail in TxCreator and txauthor before input/change callbacks.
- Every allowed template passes its exact boundary. Empty, malformed,
  OP_RETURN, P2A, unknown witness, non-standard, wrong-class, and wrong-length
  scripts return no authored result.
- Independent vector mismatches, nil entries/scripts, invalid values, overflow,
  and reported-total mismatch fail before change. Repeated sources stay aligned;
  sweepaccount decodes exact script order and rejects malformed hex.
- Actual versus estimated weight at 251/252/253 outputs, adding change across
  252->253, 252/253 inputs, multiple witness, and mixed legacy/witness classes
  is exact for modeled worst cases and never underestimated.
- Generic 2-of-3 P2WSH, large witness elements, nested scripts, Taproot script
  paths, and every CompactSize boundary compare estimates with fully signed
  serialized transactions; missing/understated descriptors fail before change.
- Failure cannot expose an externally publishable result with unwatched change.

## BW7: Transaction Invalidation

### BWR-012: Public Idempotent Branch Invalidation

#### Issue And Rationale

The SQL Store already has branch invalidation for publication and conflict
handling, but lnd's sweeper can discover an authoritative conflict involving a
channel input btcwallet does not own. The wallet cannot infer that event, while
lnd cannot call the current internal operation through a modern role.

ADR 0006 requires retained history and status transitions. Reintroducing
physical descendant removal would violate that model.

#### Detailed Fix

1. Add a narrow public operation such as
   `InvalidateUnminedTx(ctx, txid)` to `TxWriter` with wallet-owned unsupported
   and invalidation-conflict errors.
2. Gate the public Wallet role by backend capability before Store access. SQL
   uses retained-history invalidation; KV public calls return unsupported. Keep
   private publisher cleanup calling `db.Store.InvalidateUnminedTx` on both SQL
   and KV.
3. The first SQL invalidation atomically writes its generic intent plus retained
   terminal evidence:
   - one marker keyed by wallet/root with reason, semantic version,
     operation ID, branch/edge digests, and adoption flag;
   - normalized member rows with transaction ID/hash and prior/result status;
   - normalized edge rows with UTXO/outpoint, old branch spender ID/hash, and
     input index;
   - terminal status transitions and branch-owned spend-edge clears.
4. Repeated invalidation reads the retained terminal marker first. A matching
   marker is the canonical durable proof after intent compaction; a matching
   completed generic intent is only a pre-compaction fast path. Failed status
   alone never proves idempotence.
5. Replay verifies recorded members remain terminal and no active spend edge
   points to an invalidated member. A released outpoint may remain unspent or be
   owned by a later valid non-member transaction; replay never clears that later
   spender or recomputes membership from the evolved active graph.
6. Treat `reason` as immutable origin audit metadata, not request identity. An
   exact marker proves replay even when a later caller supplies another reason;
   preserve the original reason. Publisher cleanup uses the same marker path and
   is replayed, never adopted. Adoption is limited to a markerless branch with
   durable provenance from another known intent-backed conflict/rollback
   operation. Record its kind/ID in the adoption marker. Failed status alone,
   legacy provenance-free state, or branch/member/edge/replacement mismatch
   conflicts or quarantines.
7. Keep missing, confirmed, coinbase, and incompatible terminal roots as typed
   errors. Prevent SQL `CreateTx` from reviving a terminal failed transaction.
8. KV private cleanup recursively removes root and descendants plus every
   adapter-owned pending-status entry while preserving unrelated siblings and
   status rows across reopen.
9. No modern role exposes `RemoveDescendants`, physical descendant removal, or
   manual deletion. Legacy `wallet.Interface.RemoveDescendants` retains its
   walletdb/KV compatibility behavior outside the SQL role contract. S0 must
   remove unreachable SQL `Store.DeleteTx` from production interfaces or replace
   any discovered caller with retained-history invalidation.
10. Invalidation acquires exclusive cross-process wallet transaction-graph
    admission before loading the root or enumerating descendants and holds it
    through commit reconciliation. `CreateTx`, canonical/targeted ingestion,
    conflict/replacement handling, and other graph writers take shared graph
    admission; rollback/reset retain stronger exclusive database admission.
    Inside the invalidation transaction, reread/revalidate the exact
    root/member/status/spend-edge digest before writing the marker or statuses.
11. After record-before-publish is definite/reconciled, mandatory definitive
    rejection cleanup uses a bounded internal context derived without caller
    cancellation. Preserve the original publication error and join cleanup
    ambiguity. A transient/ambiguous transport or registration result retains
    the active published row for startup retry and does not invalidate.
12. Add forward migration `000017_terminal_invalidation` for retained
    marker/member/edge evidence. Never modify earlier applied migrations.
13. Before inserting/reactivating any unmined transaction, every graph writer
    checks each input hash against retained invalidation members. A match
    returns typed invalidated-ancestor conflict before transaction/intent/
    credit/edge/publication writes. Apply to direct, canonical, targeted,
    conflict/replacement, and publication reconciliation. Marker replay scans
    active unmined inputs; a post-marker descendant is integrity failure and
    quarantines. Continue allowing a valid later spender of the surviving parent
    outpoint released by invalidation.
14. Publication and startup rebroadcast hold fair shared graph admission through
    durable-attempt revalidation, bounded registration/send, and outcome record.
    Invalidation holds exclusive admission. Definitive cleanup records rejection
    while shared, releases it, then acquires exclusive without upgrading and
    rediscovers the branch. Once an exclusive waiter queues, later publishers
    cannot bypass it.

#### Primary Files

- `wallet/tx_publisher.go`
- `docs/developer/adr/0006-wtxmgr-sql-schema.md`
- `wallet/internal/db/txstore_invalidateunmined_common.go`
- `wallet/internal/db/kvdb/txstore.go` for the unsupported adapter boundary
- SQL terminal marker/member/edge schema plus status/spend-edge queries
- SQL migration `000017_terminal_invalidation` for both backends
- SQL migration `000018_publication_obligations` for both backends

#### Tests

- Invalidate a root and all descendants while preserving an unrelated sibling.
- Repeat the exact call on SQLite and PostgreSQL.
- Ambiguous committed and rolled-back invalidation outcomes reconcile by
  operation ID.
- After generic-intent GC and restart, repeat succeeds from the marker alone.
- A different root, branch set, edge set, result status, or replacement
  provenance cannot reuse a marker. A different reason on an otherwise exact
  marker succeeds and preserves the original reason.
- Publisher cleanup followed by public invalidation replays its existing marker
  without adoption. Exact markerless conflict/rollback state with durable
  provenance receives one adoption marker; status-only state is rejected.
- A non-equivalent terminal branch returns the typed invalidation conflict.
- Invalidate A, create later B spending A's released parent, retry A, and
  preserve B's active spend edge.
- KV public role returns unsupported without Store access; KV broadcast failure
  still recursively cleans root/child/grandchild and status entries.
- KV returns public unsupported and preserves existing legacy behavior.
- Missing, confirmed, replaced, and coinbase roots return the selected errors.
- A terminal SQL failed transaction cannot be revived by ordinary transaction
  ingestion.
- No test expects physical deletion.
- Legacy `RemoveDescendants` preserves its walletdb behavior while no modern
  interface exposes physical removal.
- Pause invalidation after discovery and race child/competing-spender writes
  from independent SQLite/PostgreSQL Stores. Writers block; the inverse schedule
  makes invalidation rediscover/include/conflict without an active failed child.
- Caller cancellation/deadline during blocked publish cannot cancel definitive
  cleanup. Rejection creates/reconciles one marker; transport ambiguity keeps a
  retryable active row and preserves `errors.Is` for the publication error.
- Post-marker descendants relevant through another input/output and every
  ingestion path fail with no partial state; injected descendants quarantine.
- Publication-first and invalidation-first shared/exclusive orderings, including
  startup rebroadcast and process crash, produce no stale send on both backends.

#### Exit Gate

- The operation is public, narrow, branch-scoped, and idempotent.
- Raw transaction identity, credits, and input history are not physically
  removed or rewritten. Invalidation may clear the operation-owned active
  `spent_by` pointer so the surviving parent output becomes spendable again.
  Explicit status, block-assignment, spend-pointer, and label transitions remain
  permitted only through their owning operations.

## BW8: Operation-Level Ambiguous Commit Reconciliation

### BWR-013: Reconciliation Matrix

#### Issue And Rationale

`wallet/internal/db/runtime` correctly distinguishes an ambiguous SQL COMMIT
from an ordinary transient failure. Wallet callers generally return that error
without proving whether durable state changed. Discarding a committed result can
diverge secrets, allocation counters, leases, sync state, cache publication, or
notifications. Blindly retrying can reuse keys or overwrite a newer state.

This task adds one bounded SQL mutation-intent journal because natural record
identity alone cannot make quarantine durable. Each SQL operation kind becomes
intent-enabled only with its complete registered policy/reconciler/verifier;
admission rolls out PR by PR until all reachable writers are enabled or removed.
KV does not adopt this persistence. Completed intents prune only when no
retained result or idempotence contract references them.

#### Required Quarantine ADR

Define quarantine before any implementation depends on it:

1. Persist wallet-scoped and database-scoped health rows with a monotonic
   generation, reason, affected operation/intent, and creation time.
2. Define transitions from healthy to quarantined and from quarantined to
   verified/repaired. Ordinary wallet operations can only set quarantine; they
   cannot clear it.
3. Normal Manager load fails with a typed quarantine error for an affected
   wallet. Database-scoped quarantine blocks every wallet in that Store.
4. Do not expose a partially loaded Wallet or signing/funding/read role while it
   is quarantined. Diagnostic reads and repair run only through an explicit
   maintenance API or offline tool under exclusive Store ownership.
5. Clearing requires the operation-specific deep verifier or a complete rebuild,
   an expected health generation, and an atomic compare-and-swap. An ambiguous
   clear remains quarantined until a fresh read proves the exact repaired state.
6. Concurrent Store instances check health generation before mutation and stop
   when another instance quarantines the wallet or database.
7. Define intent states precisely:
   - `applied` means COMMIT outcome/result still requires reconciliation;
   - `awaiting_side_effect` means durable state is proven and only an idempotent
     cache/chain/live-notification action remains;
    - `complete` means no post-commit work remains;
    - `quarantined` links the intent to its health record.
   `applied` is the default birth state, not universal. Foundation item 5
   permits policy-approved birth-`complete` operations with no post-commit work;
   key-only allocation is the initial case.
8. Define side-effect replay and deduplication by kind. Chain registration is
   idempotent; cache publication is reconstructed from committed state; live
   notifications deduplicate by operation ID in-process and are not replayed to
   dead subscribers after restart.
9. Define bounded retention by operation kind:
   - applied, awaiting, and quarantined intents are never pruned;
   - result-linked operations atomically externalize required proof into the
     result row/marker, then release the generic intent dependency;
   - invalidation externalizes one terminal marker/digest per root, so retaining
     that marker does not retain or pin its generic intent;
   - other complete intents prune below an active session's durable settled
     watermark and safety window, or after durable Store-session fencing.
10. Define Store-session fencing for safe compaction. Each SQL Store session has
    a random ID, monotonic operation sequence, durable active/fenced state, and
    contiguous settled/prune watermark. Reserve the next sequence, verify the
    active epoch, insert the intent, and apply the candidate mutation atomically
    while serialized against fencing.
11. Advance the settled watermark during a long-lived session across a
    contiguous sequence of complete operations with no in-flight caller. Before
    advancement, atomically externalize any retained result/idempotence proof;
    external marker rows do not pin the generic-intent watermark. Keep a bounded
    safety window and rotate the session periodically when no operation is in
    flight. GC skips retained marker rows while pruning generic intents below
    the watermark, and may also compact after durable close/fence. A stale
    process fails its epoch check instead of treating a missing intent as
    rollback. Time-based retention alone is insufficient.
12. Exempt journal control-plane writes from recursively creating intents:
    bootstrap metadata, intent state transitions, Store-session
    register/fence/watermark operations, owner-capability claim/renew/release/
    fence/handoff, outbox event claim/disposition/cursor/compacted-frontier
    transitions, health/quarantine state, and GC.
    Reconcile them through natural IDs plus expected-state/generation CAS:
    intent transitions reread operation ID/state, session writes reread
    ID/epoch, outbox writes reread wallet/sequence/delivery ID and expected
    cursor/disposition, health writes reread scope/generation, and GC
    idempotently repeats deletion only below the settled watermark. Atomic
    disposition/cursor/intent completion uses this exemption and never inserts
    a recursive intent/event. Every owner transition carries a transition ID,
    expected prior owner/epoch/state, candidate owner/epoch/state, and the
    common four-outcome fresh-read reconciliation. Closed bootstrap
    claims may begin without an existing owner capability only from exact fenced
    states: create-owner under the per-name guard either with absent wallet/
    create record for fresh creation or in adoption mode for an exact incomplete
    v1 wallet with no owner/adoption record under per-name plus exclusive Store
    fencing;
    starting-owner from completed healthy unowned; maintenance-owner from
    quarantined/maintenance-required unowned under exclusive Store ownership;
    and cutover-owner from cutover-required healthy shared state with no
    maintenance/cutover owner. Active-owner requires starting-owner. Every
    bootstrap claim remains session/health/admission fenced and reconciled.
13. Record schema, lifecycle, Manager behavior, repair authority, diagnostics,
    state transitions, retention, side-effect guarantees, and ambiguous clear
    semantics in a new ADR.
14. Enforce durable internal owner capabilities tied to identity/epoch:
    - `create-owner` performs base creation, initial-account intents, and
      completion without exposing roles;
    - `starting-owner` performs birthday repair, bounded lease cleanup, coherent
      snapshot registration, and outbox replay without exposing roles;
    - `active-owner` permits normal roles/workers;
    - `maintenance-owner` permits diagnostics/verified repair only;
    - `cutover-owner` is PR 5's database-scoped exclusive capability, available
      only in `cutover_required/cutting_over`, and coordinates participant
      proof/adoption without exposing roles. It conflicts with maintenance,
      follows database chain admission, and supports abandoned-lock takeover;
    - fenced/stopped permits only replacement-delegated reconciliation.
    Manager Load returns a healthy completed unowned Wallet. Public Start is a
    bootstrap exception: from local Stopped it obtains ownership and atomically
    claims `starting-owner`; competing Start cannot. Delivery accepts matching
    starting/active ownership; ordinary roles require active. Stop is a teardown
    exception: it may cancel/drain/fence a locally owned starting or active
    epoch and perform local-only cleanup after remote fencing. Canceled Start
    fences/releases before retry. Completion releases create-owner. KV keeps
    process-local lifecycle and creates no SQL capability/session rows.
15. Treat `awaiting_side_effect` as a wallet-wide ordered outbox. Every
    outbox-backed mutation atomically increments a transactional wallet counter
    and inserts its sequence, operation ID, stable subscription ID, immutable
    delivery ID, origin runtime epoch, kind, payload, and digest without
    rollback gaps. Dispatch records the current owner epoch separately and never
    rewrites
    delivery identity. Activation atomically reads the complete
    watch snapshot/generation/digest and freezes high-water `H` plus a bounded
    count/byte/time budget. It delivers only `sequence <= H`; arrivals above H
    remain queued for the next pass and cannot extend the current readiness
    barrier. Readiness requires contiguous terminal disposition through H and
    snapshot membership proof. A periodic fair drainer freezes a new H per pass,
    yields between budgets, and honors cancellation without a detached worker.
    Snapshot-covered registration events require verified
    `subsumed_by_snapshot` dispositions;
    other pre-frontier events need kind-specific disposition. Replay and cursor
    advancement are contiguous, and cursor/disposition/intent completion commit
    together.
    Compact terminal event/payload/disposition rows atomically only below both
    the contiguous cursor plus safety window and linked Store-session settled
    watermark/fence frontier, after externalizing required proof. Advance a
    compacted-through frontier; retain awaiting events, gaps and everything
    after the first gap. Retire fenced cursors only after successor handoff, and
    delete fenced sessions only with no intent/event/result/marker/health/audit
    reference. Missing/fenced session always means typed stale. Use bounded GC
    batches/count windows; outage-awaiting state may grow until delivery.
16. Separate PostgreSQL control capacity from the ordinary data pool. Keep
    `MaxConnections` as the data-pool limit, add a bounded admission/control
    pool, and use one connection-affine ownership session that may hold the
    Store-session and wallet-runtime advisory locks. Document the total server
    budget and close temporary bootstrap connections before Store publication.
    A protected call propagates one non-reentrant admission guard to nested
    Store work and never pins all data capacity before another connection.
17. Define one global lock order: Store-lifetime schema lease, abandoned
    ownership lock for fencing only, database admission, wallet admission,
    wallet transaction-graph admission, then data transaction/row locks.
    Normal operations take database-then-wallet shared admission; quarantine,
    repair, reset, and fencing take the required exclusive locks. A mismatch
    found under shared admission rolls back, suppresses publication, releases
    shared admission, reacquires exclusive admission, and revalidates evidence
    and health generation before quarantine CAS. Never upgrade in place.
18. Make admitted health/epoch validation the operation-start linearization
    point. PostgreSQL uses the control pool/session above; SQLite uses the
    stable persistent DB-adjacent shared/exclusive and ownership locks from
    BWR-000.
    `WriteGate` remains process-local fairness only. Ownership locks, never
    heartbeat timeout, prove liveness. Fencing acquires an abandoned ownership
    lock, then admission, then CASes active to fenced. A paused owner retains
    its lock; a crashed owner releases it. Losing the local ownership session
    immediately invalidates the runtime latch.
19. Derive every PostgreSQL advisory key on its pinned connection from a
    length-prefixed protocol version, server-returned `current_database()`,
    btcwallet schema, and lock kind. The schema key contains no wallet,
    session, create, or process identity and is identical for bootstrap/
    migration, lifetime lease, and data-transaction fencing. Admission,
    ownership, arbitration, maintenance, and lease-guard keys append only the
    durable scope identity declared for that lock kind. Never use DSN, host,
    port, user, TLS, password, pooler, or process-registry identity. Aliases for
    one server/database contend; distinct databases do not.
20. Add cross-process wallet transaction-graph admission. Creation, ingestion,
    and conflict/replacement handling take shared graph admission; invalidation
    takes exclusive graph admission; reset/rollback use stronger database
    exclusion. Never upgrade in place.
21. Land quarantine with usable repair authority. PR 3 provides a non-role
    `wallet/maintenance` facade, a closed internal verifier registry, offline
    `cmd/btcwalletsql`, exclusive ownership/admission, expected
    health-generation CAS, exact before/after digests, durable redacted audit
    output, and ambiguous
    clear reconciliation. Missing verifier is diagnose-only. Later operation PRs
    register verifiers before they can create quarantine.

#### Foundation Work

1. Add a public wallet ambiguity error that does not expose runtime internals.
2. Add focused internal helpers only for repeated classify/read/compare
   mechanics. Operation-specific candidate comparisons stay with the owning
   workflow.
3. Turn the S0 mutator inventory into a checked matrix. Every reachable Store
   mutation records candidate facts, old-state facts, caller identity, replay
   policy, side effects, owning PR, and failure-injection coverage. Explicitly
   remove operations proved unreachable rather than leaving an unowned row.
4. Add a generic mutation-intent table with operation ID, Store session ID,
   monotonic session sequence, wallet/database scope, operation kind,
   expected-state digest, candidate/result digest, state, and timestamps.
   Operation-specific tables or result rows may reference the ID when more
   facts are required.
    Land health, intents, sessions, runtime epochs, outbox, watch generation,
    account-watch state/subscription/current-delivery identity, repair audit,
    and canonical-state foundation in forward migration
    `000012_runtime_foundation`.
5. Make mutation-intent admission opt-in by closed operation kind. A compiled
   policy registry supplies payload version/decoder, candidate/old facts,
   verifier/reconciler, side-effect policy, quarantine scope, and both failure
    seams before enabling a kind. PR 3 initially enables base wallet create,
    existing-shape sequential/imported initial account create, v1 adoption,
    incomplete-create abort/tombstone, completion, birthday repair, and startup
    lease cleanup, plus the closed owner-transition control-plane policies
    above.
   Later PRs register then enable their kinds. Missing components fail before
   mutation; unknown/unsupported applied payloads diagnose/fail closed and never
   generic-complete. Every writer has an S0 row whether enabled or disabled;
   disabled writers cannot insert intents. For enabled writes,
   session/capability validation, sequence, intent/outbox, and mutation commit
   atomically. A
   policy-approved no-side-effect operation may be born complete; key-only
   allocation is the initial case.
6. After durable reconciliation, move operations with pending external work to
   `awaiting_side_effect`; operations without it move directly to `complete`.
7. Replay awaiting side effects by operation ID without deep-verifying the
   mutation again. A chain-backend outage may grow this queue but does not block
   unrelated mutations or convert it into corruption work.
8. After cache publication or an idempotent external side effect, mark the
   intent `complete` with expected-state CAS. For outbox work, event
   disposition, contiguous runtime cursor advance, and completion commit
   atomically.
   Reconcile an ambiguous completion update by rereading durable state; never
   erase the only evidence.
9. Keep applied/unresolved intents durable across crash and restart. Startup and
   concurrent writers invoke targeted verification only for known enabled kinds.
   Unknown payloads remain diagnosis-only; disabled writers produce no intent.
10. Implement active-session settled-watermark advancement, bounded safety
    retention, idle rotation, fencing, terminal outbox/payload/disposition
    compaction, cursor handoff/removal, fenced-session reference checks/removal,
    compacted-through frontiers, and idempotent control-plane CAS without
    recursively journaling those writes. Missing session is always stale.
11. Implement the accepted quarantine ADR for an atomic operation whose state is
   partial or cannot be reconciled. Use wallet-scoped quarantine for isolated
   wallet state and database-scoped quarantine for shared chain/schema state.
12. A runtime latch mirrors durable quarantine but cannot clear it. Stop/load is
   not repair, and normal Manager load returns the typed quarantine error.
13. Keep ordinary open checks constant-cost: schema/genesis metadata, migration
   state, wallet manifest shape, health generation, quarantine flags, and
   applied-intent counts. Awaiting-side-effect counts feed the replay worker,
   not deep verification.
14. Run targeted deep verification for account/address, transaction/spend-edge,
    ancestry, or lease invariants only for an applied intent, quarantine,
    explicit repair, or dedicated integrity command. Do not scan complete
    history on every open.
15. Every protected operation validates Store session, owner capability/epoch,
    health generation, and applied intents while admitted so stale processes
    stop and concurrent Stores observe quarantine. Awaiting-side-effect intents
    do not block unrelated mutations.
16. If durable quarantine itself cannot be recorded, keep the affected Store
    closed. The applied intent remains durable, so a fresh process or another
    Store instance runs its targeted verifier before mutation; a process-local
    latch is never the only evidence.
17. Keep runtime caches unpublished until reconciliation succeeds. The current
   pass-through cache needs no data migration, but tests must preserve this
   ordering for future caches.
18. Emit notifications or invoke chain side effects only from a confirmed
    durable result.
19. Add runtime-epoch/outbox schema and generic startup replay in PR 3. An
    operation-specific PR supplies its payload encoder and delivery action, but
    cannot invent another completion mechanism.
20. Move health/session admission into each backend's transaction boundary and
    add protected-operation admission to Controller, signer, funding, key, and
    role read entry points in PR 3. Include the PostgreSQL control pool,
    connection-affine ownership executor, propagated guards, and global lock
    ordering. A pre-transaction process-local check is not a correctness
    boundary.
21. Implement ownership-lock registration, renewal diagnostics, graceful close,
    crash fencing, admission frontier, sequence reservation, rotation,
    watermarking, event/cursor/session compaction, reference-safe GC, and stale
    classification in PR 3. Fencing/GC serialize against any operation still
    able to commit, deliver, or reconcile.
22. PR 3 implements maintenance diagnosis/repair, generic verifiers, audit/CAS,
    and command skeleton. PRs 4, 5, 6, and 7 register operation verifiers; PR 8
    owns command-level and cross-process acceptance.
23. PR 3 owns the coherent exact-script snapshot/frontier read and
    starting-owner activation protocol. PR 5 consumes and extends it for
    canonical proof.
24. Exempt publication from the registration cursor but give every modern
    broadcast an ownership-independent durable attempt in forward migration
    `000018_publication_obligations`. Store operation/wallet ID, canonical full
    transaction serialization including witness, serialization digest, derived
    and verified txid/wtxid, label, optional wallet-history link, state, typed
    outcome, attempts, and disposition. Persist and reconcile this attempt
    before every backend operation, including mempool/status/acceptance probes,
    registration, or send. Wallet-owned transactions may atomically link
    history; ownership-free/key-only attempts create no credits/UTXOs/balance/
    watch ownership. Startup reconstructs only from retained canonical bytes and
    retries nonterminal attempts in dependency order.
    Persist explicit parent-attempt edges derived from those bytes at enqueue
    time, including ownership-free parents; reject cycles/conflicting edge
    rewrites. Startup/topological retry sends a child only after each dependency
    is confirmed, accepted/already-known in the current retry epoch, or proven
    backend-known.
    Accepted/already-known moves to `accepted_unconfirmed` and remains a durable
    rebroadcast obligation across restart/backend eviction until confirmation,
    retained definitive consensus-invalid rejection, or marker-backed wallet
    invalidation/replacement. Confirmation suppresses send while canonical and a
    reorg reopens the obligation; compaction waits for the stated confirmation
    retention depth. Fee/policy/non-final/mempool/timeout conditions are never a
    terminal policy.
    Ownership-free confirmation/reorg transitions consume PR 2's typed status
    proof and bind it to the current canonical/backend generation. Unsupported
    or unprovable status after enqueue retains the obligation and blocks
    compaction; it never fabricates confirmation.
    Retryable, unknown, or not-attempted also retains. Normalize cancellation
    before dispatch to not-attempted and unacknowledged post-dispatch
    cancellation to unknown. Definitive rejection invalidates only a linked
    active wallet transaction; otherwise terminal-reject. Compaction requires
    terminal state,
    retention depth/time, no descendant/dependency/history/intent reference, and
    externalized confirmation/invalidation proof. It never blocks registration.
    Under exclusive graph admission, terminal parent rejection/invalidation
    transitively records `blocked_by_terminal_parent`. Marker-invalidate linked
    active wallet descendants and terminally dispose ownership-free descendants;
    retain the parent proof and dependency edges until descendant audit and
    compaction requirements are satisfied.
25. Make typed mutation admission the only SQL semantic-write choke point:
    `Store.Mutate(ctx, MutationPlan)` validates owner/session/health, reserves
    sequence, inserts intent/outbox, and commits candidate CAS. Backend query
    handles and generated write methods remain private to this executor. A
    closed allowlist covers only bootstrap/migration and item-12 control-plane
    CAS. Add repository AST/static checks that reject any semantic write outside
    the choke point and fail when a new sqlc mutation lacks a policy/S0 row.
26. Repair/clear is reference-releasing, not only state-changing. In the same
    audited CAS that proves repair, externalize the final verifier digest, move
    or complete the linked intent/event/marker, remove obsolete health/audit
    references, and recompute contiguous settled/cursor eligibility. GC walks a
    declared foreign/logical reference graph and deletes nothing while any live
    operation, event, result, marker, health, audit, dependency, or session row
    references it.

The public error, intent lifecycle, health generation, quarantine state, and
failure-injection seams land before any PR that depends on them. Every later PR
must ship reconciliation for each mutation it changes; the final BW8 PR closes
only unchanged pre-existing mutations from the S0 inventory.

#### Operation Policies

Unless a policy below says otherwise, an absent intent proves rollback only
under the Store-session fencing, active-epoch, and prune-watermark rule from the
common ambiguity boundary.

**Initial wallet creation - PR 3**

- Before applying v2 prefix rules, adopt every incomplete PR-1/v1 wallet once
  under create-owner fencing. Verify exact wallet/secrets/sync facts, the v1
  manifest, and every existing account as a required exact v1 member; reject
  extras, mismatches, duplicate identities, or impossible base facts. Because
  legacy execution order was not retained, accept any exact required-account
  subset reachable at a transaction boundary, not only a sorted prefix.
- One adoption transaction freezes a v2 effective plan with verified adopted
  rows first in deterministic durable-identity order, assigns base/account
  operation IDs and manifest links, records request/effective/adoption digests,
  and advances adoption state. Reconcile ambiguous adoption from the retained
  candidate IDs/digest. No role publishes before adoption and normal completion
  succeed; subsequent writes follow only v2 prefix rules.
- The base `Store.CreateWallet` intent owns only its wallet row/ID, secrets,
  initial birthday/sync state, create manifest, base operation ID, and
  `create_complete=false`. Schema/genesis is a prerequisite; initial accounts
  and completion belong to later transactions.
- Each default/shell account has its own account-create operation ID and
  prepared account intent. The manifest defines the required set but cannot
  attribute another transaction's account to the base create. Watched initial
  accounts may queue registration for the eventual starting owner.
- Completion verifies every manifest-required account/operation ID and CASes
  the exact manifest plus `create_complete=false` to true under its own
  operation ID. Reconcile ambiguity by wallet ID, manifest, required account
  set, and completion ID. Publish only after definite/reconciled completion.
- Exact committed prefixes are healthy: no rows; base wallet/secrets/sync/
  manifest with completion false; then a deterministic prefix of complete
  manifest account operations; then all accounts with completion false; then
  completed true. Matching create-owner retry reconciles unresolved exact work
  and resumes the first absent step without maintenance. Quarantine only
  impossible intra-step partial facts, wrong operation links, out-of-order/
  mismatched manifest facts, or an unprovable atomic result. Different manifest
  or passphrase rejects without quarantining a healthy prefix. Publish only the
  completed prefix; later PRs may extend fields without moving transaction
  ownership.
- Persist `create_state`, request/effective-plan digests, owner ID/epoch, last
  complete step, and timestamps in migration 12. Startup and Manager listing
  classify every `create_complete=false` row before cutover: exact prefix is
  resumable, an in-flight ambiguous step remains resolving, and impossible or
  unprovable facts become wallet-scoped quarantined with evidence. Only the same
  request digest and authenticated create credentials may resume; owner takeover
  uses the common stale-owner fencing protocol.
- Incomplete rows are never returned by ordinary wallet/open/account APIs and do
  not contribute to canonical-frontier or activation required-wallet sets. A
  completion racing activation is fenced behind Store ownership, then joins as
  a new candidate and must register before serving. Wallet-scoped quarantine
  does not block unrelated complete wallets unless it proves a shared invariant
  unsafe.
- Cleanup is explicit or bounded maintenance, never open-time deletion. Under
  Store write ownership, `AbortIncompleteCreate` verifies request digest, stale
  create owner, no unresolved intent/lease/reference, and an exact healthy
  prefix, then removes it atomically with a tombstone operation. Quarantined or
  ambiguity-bearing prefixes require repair proof. Tests cover every prefix,
  crash/restart, stale takeover, activation races, resume, abort, and
  visibility.
- PR-1 fixtures cover every create crash point and differently ordered v1
  initial-account subset. Exact adoption resumes after both ambiguous outcomes;
  extra/mismatched rows quarantine without attributing them to the create.

**Transaction creation**

- Compare the complete serialized transaction, including witness data. Txid
  equality alone is insufficient because txid excludes witness.
- Exact duplicates replay credit and spend-edge repair idempotently.
- Persist/reconcile canonical full transaction bytes and the ownership-
  independent publication attempt before any mempool/status probe,
  registration, or send. Derive/verify txid, wtxid, and parent edges from those
  bytes; same txid with different complete serialization conflicts.
- Publication/rebroadcast acquires fair shared graph admission, rereads exact
  attempt bytes, active linked-wallet status, marker membership, health, and
  runtime epoch, then holds admission through bounded typed registration,
  publication, and durable outcome. Invalidation waits exclusively; a terminal
  row prevents later send. Exclusive waiters block newer shared attempts.
- Accepted/already-known records the attempt result but keeps the publication
  obligation `accepted_unconfirmed`; retryable/unknown/not-attempted also
  retains. Confirmation or another authoritative terminal condition completes.
  Definitive rejection records outcome while shared, releases it, then acquires
  exclusive admission without upgrade, rediscovers the branch, and marks only a
  linked active wallet transaction. Ownership-free rejection only terminates
  the attempt. Parent-edge retention and topological retry follow foundation
  item 24.
- Startup registers the coherent snapshot before dependency-sorted parent/child
  rebroadcast. A failed publication never blocks later registration outbox
  events.
- First-call already-known still creates the durable attempt, records
  `already_known`, and remains an accepted-unconfirmed obligation across
  restart. Terminal parent disposition propagates transitively under exclusive
  graph admission before any descendant can send.

**Sync and scan batches**

- Replay the complete batch.
- Duplicate transactions are reparative observations.
- Horizon extension remains monotonic.
- Synced-tip updates use the expected-tip, contiguous-ancestry, and stale-batch
  rules from BWR-005.

**Wallet secrets**

- Capture the complete candidate ciphertext and expected old state.
- Use compare-and-swap so a concurrent rotation cannot be overwritten.
- Candidate match means committed; old-state match permits one replay; any
  third state is a conflict or unknown result.
- Publish unlocked/locked secret cache state only after proof.
- Cover private-passphrase rotation and prove the selected passphrase after
  restart for committed and rolled-back outcomes.

**Account creation**

- Link the created account to the generic mutation operation ID.
- Reconcile by operation ID, then compare scope, number, name, XPub, provenance,
  schema, fingerprint, discovery mode, and encrypted secret identity.
- Missing intent plus unchanged old state permits a new attempt. A later
  caller's matching deterministic account is not attributed to this caller.
- Active watch-chain creation links one horizon-registration payload and returns
  readiness only after typed delivery. Create-owner accounts defer the same
  payload to starting-owner snapshot/replay; an empty zero-window snapshot may
  be ready, and its first exact allocation establishes receiving authority only
  after delivery. Key-only creates no event and is born complete.

**Address allocation**

- Reconcile by the allocation operation ID from BWR-007.
- Compare the complete requested batch by locator, script, public key, type,
  account, and discovery mode only after the intent matches.
- Matching intent and complete batch returns success; no intent allocates a
  fresh range; partial or mismatched result quarantines the wallet.
- Never call chain registration before this decision.

**Imports and tapscripts**

- PR 1 updates ADR 0005: imports never start historical rescans, while a public
  live-watched import may require bounded registration after persistence before
  returning usable state. Store-only/offline import remains database-only.
- Compare the complete imported identity, encrypted candidate material, wallet
  mode, and expected absence.
- A matching candidate may publish address/script lookup state. A different
  import at the same script is a conflict.
- PR 4 atomically stores the complete watched raw-address/script/tapscript
  registration payload in the runtime outbox. Return a usable result only after
  idempotent delivery; failure leaves one repairable awaiting event. Imported
  XPub account creation remains under account policy, and key-only accounts
  reject receiving imports.

**Account rename - PR 7**

- Capture stable account identity, scope, old/new names, and both name-index
  facts. Candidate proof is the exact new row/index and absent old index.
- Rollback proof is the exact old row/index and unused new name. ABA, changed
  target identity, or another account owning the new name conflicts; impossible
  partial index state quarantines. Rename has no external side effect.

**Birthday repair - PR 3**

- Capture wallet ID, manifest, persisted birthday/sync facts, and the exact
  candidate repair before mutation. Candidate match commits; exact old facts
  permit replay; competing or partial state fails closed. Starting-owner cannot
  proceed to snapshot activation until reconciliation completes.

**Wallet metadata, labels, and transaction updates - PR 7**

- Use expected-old-value compare-and-swap for single-row transitions.
- Label and transaction metadata replay must preserve allowed overwrite rules.

**Leases**

- Reconcile with the same unique lock ID and attempted expiration.
- Same-owner acquisition or renewal is success.
- Release records outpoint and expected owner in the mutation intent atomically
  with deletion.
- A normal release of an already absent lease is a harmless idempotent success;
  a present lease owned by someone else remains an error.
- A different owner remains a conflict; do not release it during rollback.

**Manager create completion - PR 3**

- Reread wallet ID, create manifest, and `CreateComplete` after an ambiguous
  completion update.
- Verify every required initial-account operation ID before the completion CAS.
- A matching completed wallet is published rather than discarded.
- A mismatched manifest fails closed.

**Rewind and rollback - PR 5**

- Serialize through the syncer and capture the expected source tip and complete
  target tip.
- Reconcile transaction status, block assignment, UTXO/spend effects, recovery
  state, and terminal tip as one atomic result. Terminal-tip equality alone is
  insufficient.
- Do not process a later chain event until reconciliation finishes.

**Database-wide chain reset - PR 5**

- Use one database-scoped operation ID for the coordinator reset generation and
  every affected wallet. Capture the source shared-block digest and each
  wallet's complete old chain-state digest.
- Candidate proof covers preserved genesis, removal of later blocks, every
  wallet's sync/birthday references, transaction status/block assignment,
  recovery state, UTXOs, and spend edges. Partial reset quarantines the
  database.
- Hold exclusive database admission through reconciliation. Cancel/mark syncers
  unsynced before mutation, but do not rebuild or restart them before the exact
  committed or rolled-back outcome is proven.

**Store-only transaction deletion - PR 7**

- S0 must confirm SQL `DeleteTx` has no production caller, then remove it from
  the production Store contract and adapters. Corruption tests use an explicit
  test-only fixture seam.
- If S0 finds a caller, replace it with retained-history invalidation. Do not
  make physical deletion public or give it a new normal-operation contract.

**Startup lease cleanup - PR 3**

- Capture one expiration cutoff and replay with the same cutoff.
- Deleting the same expired lease set is idempotent; cleanup cannot remove a
  lease renewed beyond the captured cutoff.

**Transaction invalidation**

- Reconcile through the retained BWR-012 marker and operation payload. Every
  member must have its exact recorded result status, not merely any terminal
  status.
- For each recorded edge, require only that an invalidated member no longer owns
  it. The current spender may be nil or a later valid non-member. Never clear,
  reject, or overwrite that later spender.

#### Primary Files

- `wallet/internal/db/runtime/runtime.go`
- the new quarantine ADR and health/mutation-intent SQL schema
- `wallet/internal/db/pg/runtime.go`
- `wallet/internal/db/{sqlite,pg}/store.go`
- `wallet/runtime_store_registry.go`
- `wallet/maintenance/`
- `wallet/internal/db/maintenance/`
- `cmd/btcwalletsql/main.go`
- `wallet/manager.go`
- `wallet/internal/db/walletstore_updatewalletsecrets.go`
- `wallet/internal/db/accountstore_createderived.go`
- `wallet/internal/db/addressstore_newderived.go`
- `wallet/internal/db/{sqlite,pg}/utxostore_leaseoutput.go`
- `wallet/internal/db/{sqlite,pg}/utxostore_listleasedoutputs.go`
- `wallet/internal/db/txstore_common.go`
- `wallet/internal/db/{sqlite,pg}/txstore_batch.go`
- `wallet/internal/db/{sqlite,pg}/txstore_applyscanbatch.go`
- `wallet/internal/db/{sqlite,pg}/txstore_rollback.go`
- `wallet/syncer.go`
- `wallet/reset_coordinator.go`
- `wallet/tx_publisher.go`

#### Failure-Injection Tests

For each operation, force both ambiguous outcomes:

1. COMMIT happened but the caller received an error.
2. COMMIT did not happen and the caller received the same error class.

Also test:

- A competing write between ambiguity and reconciliation.
- Reconciliation read failure.
- Partial durable state injected through a test-only seam.
- Applied intent survives process restart and is reconciled before mutation.
- A crash after side-effect invocation but before cursor/completion COMMIT may
  invoke idempotent registration again. It produces one observable watched
  state, one durable disposition, and no duplicate in-process live notification
  where operation-ID deduplication applies.
- A chain outage accumulates `awaiting_side_effect` intents without triggering
  deep verification or blocking unrelated mutations.
- Continuous concurrent producers above frozen high-water H cannot starve
  activation/scan completion through H; bounded fair passes eventually deliver
  later events, and cancellation leaves a resumable contiguous cursor.
- Quarantine-write failure remains detectable after restart and from a second
  concurrently open Store instance.
- No cache publication, notification, chain registration, signing, or broadcast
  before proof.
- Import, tapscript, password-change, label, birthday, rollback, rewind, and
  startup lease-cleanup outcomes from the S0 mutator inventory.
- Stop/load cannot clear quarantine; verified repair or rebuild can.
- Constant-cost open checks run for healthy Stores; deep verification is
  triggered only by applied intents, quarantine, maintenance, or explicit
  integrity requests.
- Retention/compaction tests keep applied, awaiting, quarantined, result-linked,
  and terminal-invalidation evidence according to the ADR while bounding
  prunable complete-intent growth.
- Deliver over twice the safety window, rotate/fence repeatedly, and prove
  bounded terminal events/payloads/dispositions/cursors/sessions. Cursor or
  settled-watermark lag, awaiting/gaps, and result/marker references block GC;
  successor handoff/reference release permits it. GC ambiguity/races recover,
  and a compacted old session is always stale.
- Repair clear atomically externalizes proof and releases every obsolete
  health/audit/intent/event reference; GC remains blocked for each independently
  injected reference, then advances immediately after the final safe release.
- A delayed caller from an active session can reconcile before pruning; a
  closed/fenced session cannot resume after its complete intents compact.
- ABA state after compaction cannot turn an absent intent into rollback proof.
- A long-lived session advances a contiguous settled watermark and bounds
  complete-intent growth without waiting for Store close.
- Sequence reservation/candidate mutation races with fencing produce one winner
  and no unfenced mutation.
- Ambiguous intent-state, session, quarantine, and GC control-plane writes
  reconcile by their natural ID/generation without recursive intents.
- PostgreSQL `MaxConnections=1` still permits a protected Store transaction
  while ownership is pinned in the control plane. Admission-pool saturation and
  canceled waiters leak no connection or lock.
- Concurrent quarantine lets an operation admitted before its linearization
  finish and rejects one starting after, including a signer with cached key
  material. Discovery under shared admission releases and reacquires exclusive
  admission rather than upgrading.
- Loss of an ownership session permits fencing and runtime replacement. A
  paused owner retaining its lock cannot be fenced. Old-epoch workers and cursor
  gaps cannot mutate, deliver out of order, or advance the cursor.
- Ambiguous claim/release/fence/handoff for every owner capability reconciles to
  one owner/epoch or conservative non-serving state. Successor replay keeps the
  same delivery ID with a new dispatch epoch; delayed predecessor ACK/NACK
  cannot duplicate delivery or advance its cursor.
- Initial wallet creation, account rename, multi-wallet ResetChain, watched
  import/tapscript delivery, and every remaining S0 matrix row cover committed,
  rolled-back, competing-write, partial-state, and restart outcomes.
- Base-wallet, each initial account, and completion crash/ambiguity resume only
  their owned step. No base intent claims later account/completion facts.
- Every valid staged-create prefix resumes without maintenance; injected
  intra-step corruption quarantines. Competing Start, Stop-during-start,
  canceled retry, abandoned starting takeover, remote fencing, and KV
  pass-through follow the lifecycle matrix.
- Registry enumeration rejects writers without S0 rows, enabled kinds missing
  policy components, disabled kinds inserting intents, and unknown applied kinds
  being treated as complete.
- Static mutation audit fails on a direct sqlc semantic write, an unallowlisted
  control write, and a new generated mutation without policy; all production
  semantic paths pass only through `Store.Mutate`.
- PostgreSQL URL/keyword, host, role, pooler/direct aliases contend on every
  server-derived lock kind; different databases do not contend or leak DSNs.
- Diagnose is non-mutating; active runtime blocks maintenance; wrong generation,
  mismatch, or missing verifier cannot clear. Exact and ambiguous repair audit
  correctly across restart without secrets/DSNs.
- Ownership-free/key-only publication crash points around record/status probe/
  register/send/ack replay through the attempt queue. With no wallet-history
  fallback, restart reconstructs exact full bytes and IDs. First-call
  already-known persists before its probe and is idempotent; serialization
  variants conflict. Capability-false rejects ownership-free enqueue with no
  attempt; advertised support followed by unsupported retains the attempt and
  fences the backend generation.
- Accepted-unconfirmed survives restart and simulated mempool eviction and
  rebroadcasts. Parent-before-child retry, external parent proof, cycles,
  rejection/invalidation, confirmation, and every retention reference prove no
  child sends early and no obligation compacts while still authoritative.
- Terminal parent rejection/invalidation marks every transitive descendant
  `blocked_by_terminal_parent`; linked wallet descendants invalidate and
  ownership-free descendants terminally dispose without unbounded graph pins.

#### Exit Gate

- Every wallet mutation in the frozen S0 inventory has an implemented and
  failure-injected ambiguous-commit policy.
- No ambiguous result can reuse a committed key locator or release another
  owner's lease.
- No external side effect is emitted from an unproven outcome.

## BW9: Backend And Chain Evidence

### BWR-014: Complete Wallet-Level Proof

#### Issue And Rationale

Store conformance tests prove queries and transactional workflows, but they do
not prove Controller lifecycle, live chain ordering, chain registration,
funding, publication, restart, or reorg behavior. A native SQLite
create/start/sync test now exists, so the older claim of no native lifecycle
test is stale; the complete flow and PostgreSQL parity remain absent.

#### Required Test Layers

**Shared unit tests**

- Public type and error conversions.
- Controller state transitions.
- Vault script semantics.
- PSBT merge/sort behavior.
- Operation-level ambiguity classifiers.

**Store conformance**

- Every changed semantic operation runs against SQLite and PostgreSQL in
  `wallet/internal/db/itest`.
- KV unit tests cover every shared contract changed by this task.

**Native SQL chain scenario**

Parameterize one maintained btcd scenario over SQLite and PostgreSQL and prove
this complete sequence without a KV sidecar:

```text
create -> start -> unlock -> derive receiving address -> receive
    -> confirm -> change passphrase -> restart with new passphrase
    -> recover -> allocate exact key-only family -> fund -> sign
    -> publish -> confirm -> reorg -> reconfirm -> stop
```

The scenario also covers:

- Watch-only create/unlock/restart.
- Tapscript import and lookup.
- Existing-unused registration retry.
- PSBT metadata and duplicate outputs.
- Persisted nonstandard coin scope signing.
- Change registration before result return.
- Sweeper-style branch invalidation.
- Ambiguous commit failure injection at representative lifecycle, allocation,
  transaction, and sync operations.

**Additional backend evidence**

- Extend the integration harness to provision and select a native PostgreSQL
  wallet Store. Remove the current explicit PostgreSQL rejection for this
  named scenario.
- Run the complete native SQL scenario above on PostgreSQL, not a reduced smoke
  path.
- Run SQLite canonical-event and address-registration vectors on bitcoind and
  neutrino after the parameterized btcd gate.

**Required CI**

- Register stable integration cases in `itest/list_on_test.go`: `wallet sql
  readiness`, `syncer canonical events`, and `address registration repair`.
- Add non-matrix jobs `sql-role-btcd-sqlite`, `sql-role-btcd-postgres`,
  `sql-role-bitcoind-sqlite`, `sql-role-neutrino-sqlite`, and
  `sql-role-cross-process-locks`.
- The lock job runs stable bootstrap/runtime cross-process selectors against
  SQLite and PostgreSQL, including pool-size-one and filesystem-alias vectors.
  It runs `make itest-sql-role-cross-process-locks`, which invokes
  `make itest-db-json-assert db=<backend>
  expected=TestSQLRoleCrossProcessLocks` once per backend and fails if either
  exact test does not run and pass.
- Add aggregate job ID `sql-role-readiness`, display name `SQL role readiness`,
  with `if: always()`. S0 freezes actual existing job IDs: `lint-check`,
  `check-commits`, `unit-test`, `itest-db`, `itest-btcd`, `itest-neutrino`,
  `itest-bitcoind`, and `finish`. Its literal `needs` is that static union plus
  all five SQL-role jobs. `lint-check` invokes
  `scripts/run-required-static-checks.sh`, containing format, module, RPC/SQL
  generation, build, proto lint, and Go lint instead of pretending those steps
  are independent jobs.
- `scripts/assert-ci-needs.sh` receives `github.event_name` and `toJSON(needs)`,
  verifies the exact key set, and applies event-specific results. On
  `pull_request`, every need including `check-commits` must be `success`. On
  `push`, `check-commits` must be `skipped` and every other need must be
  `success`. Any other skip, missing/extra/renamed key, cancellation, failure,
  or event fails. Branch protection requires this stable aggregate name.
- Every named itest job records `go test -json` output and fails unless its
  expected sanitized subtest reports both run and pass. Top-level
  `TestBtcWallet` success with zero selected cases is a CI failure.
- Add `make itest-json-assert`, backed by
  `scripts/run-itest-json-assert.sh`. It accepts exactly one sanitized full
  subtest name, invokes `go test -json -count=1` with an anchored selector, and
  requires that exact test's run/pass events. CI and final verification use it;
  E2E jobs use it. Add package-aware `make itest-db-json-assert`, backed by the
  corresponding script mode: select `wallet/internal/db/itest`, apply the
  backend build tags from `make/testing_flags.mk`, use anchored `-run` and
  `-count=1`, and require exact run/pass JSON events. The cross-process target
  applies this DB wrapper separately to SQLite and PostgreSQL.
- External evidence owner is the `btcsuite/btcwallet` repository administrator
  performing the PR 8 merge. Record their login, update the active ruleset for
  the default/master branch to require exact context `SQL role readiness`, and
  run checked `scripts/assert-required-context.sh --repo=btcsuite/btcwallet
  --context='SQL role readiness' --sha=<evidence SHA> --output=<artifact>`.
  The script obtains `.default_branch`, enumerates
  `GET /repos/{owner}/{repo}/rulesets`, fetches each ruleset detail, and uses
  `jq -e` to require one object with `.enforcement == "active"`, a ref include
  of `~DEFAULT_BRANCH` or `refs/heads/<default>`, no matching exclusion, and a
  `.rules[]` object whose `.type == "required_status_checks"` and whose
  `.parameters.required_status_checks[]?.context` equals the exact context.
  Given the recorded evidence SHA, fetch authenticated check runs and require
  exactly one completed/successful `SQL role readiness` run; record its producer
  GitHub App ID. Require the matched ruleset entry's non-null `integration_id`
  to equal that producer ID. Emit minimal matched JSON (SHA/check URL/name/
  conclusion/app ID plus ruleset ID/name/enforcement/ref/rule/context/
  integration ID) and exit nonzero on zero/ambiguous/mismatched results. Attach
  authenticated raw check-run and ruleset list/detail responses;
  YAML/screenshots are insufficient.

**Regression and race evidence**

- KV Manager and wallet unit tests for shared code.
- Wallet race tests around lifecycle, allocation, leases, and sync.
- Re-run the fair 1,000-block, 10,000-block, and ten-hit/1,000-block sync
  workloads with ten independent samples. Reject a statistically significant
  regression above 10 percent in wall time or allocations per block unless the
  PR documents and separately approves the tradeoff.
  Freeze names `BenchmarkFairSQLiteSync/Backlog/Empty/Blocks-1000`,
  `BenchmarkFairSQLiteSync/Backlog/Empty/Blocks-10000`, and
  `BenchmarkFairSQLiteSync/Backlog/Sparse10/Blocks-1000`.
- S0 records immutable `S0_SYNC_BASE_SHA` and a benchmark-only
  `SYNC_HARNESS_SHA`. The benchmark script applies the identical harness commit
  to clean detached base/candidate worktrees when absent, verifies production
  source hashes remain those of each measured revision, and requires the
  discovered benchmark-name set on each side to equal exactly the three frozen
  names. Zero/missing/extra names fail before benchstat; archive both lists,
  SHAs, apply logs, and raw samples.
- Maintain these literal allocation benchmark names:

```text
BenchmarkNewAddressSQLBaseline/SQLite/WatchChain/Count-1
BenchmarkNewAddressSQLBaseline/PostgreSQL/WatchChain/Count-1
BenchmarkNewBulkAddressesAPI/SQLite/KeyOnly/Count-1
BenchmarkNewBulkAddressesAPI/SQLite/KeyOnly/Count-32
BenchmarkNewBulkAddressesAPI/SQLite/KeyOnly/Sequential-32xCount-1
BenchmarkNewBulkAddressesAPI/PostgreSQL/KeyOnly/Count-1
BenchmarkNewBulkAddressesAPI/PostgreSQL/KeyOnly/Count-32
BenchmarkNewBulkAddressesAPI/PostgreSQL/KeyOnly/Sequential-32xCount-1
BenchmarkNewBulkAddressesAPI/SQLite/WatchChain/Count-1
BenchmarkNewBulkAddressesAPI/SQLite/WatchChain/Count-32
BenchmarkNewBulkAddressesAPI/SQLite/WatchChain/Sequential-32xCount-1
BenchmarkNewBulkAddressesAPI/PostgreSQL/WatchChain/Count-1
BenchmarkNewBulkAddressesAPI/PostgreSQL/WatchChain/Count-32
BenchmarkNewBulkAddressesAPI/PostgreSQL/WatchChain/Sequential-32xCount-1
BenchmarkSQLRoleAllocationActivation/SQLite
BenchmarkSQLRoleAllocationActivation/PostgreSQL
```

- Use `N=32`, `-benchtime=1x -count=10`, a fresh database/wallet per sample,
  fixed seed/account/external branch/script class, and no invalid children.
  Create/activate before timing and reset SQL counters. Watched timing ends
  after synchronous delivery; report activation separately.
- In PR 6, land the benchmark harness and exactly the two
  `BenchmarkNewAddressSQLBaseline` names in an immutable baseline commit; record
  that commit as `ALLOCATION_BASE_SHA`. Put their complete harness/body in a
  dedicated baseline file. Later PR 6 commits add the twelve bulk names and two
  activation names in a separate candidate file. The runner requires its
  SHA-256 to equal the same baseline file's SHA-256 at `ALLOCATION_BASE_SHA` and
  rejects changes before execution.
  It also requires the base discovered set to equal exactly those two names and
  candidate set to equal exactly all sixteen literal names above, failing on
  zero/missing/extra matches. The base watched count-one path is the
  conservative denominator for both new modes. Define `C(name)` as the
  arithmetic mean of the
  ten raw `ns/op` samples for that literal benchmark. Per backend, `Base1`,
  `KeyOnly1`, `WatchChain1`, `Count32`, and `Sequential32` mean the matching
  literal names above. Require
  `C(KeyOnly1)/C(Base1) <= 2.5` and
  `C(WatchChain1)/C(Base1) <= 3.5` per backend.
- Preserve raw benchmark files, then create comparison-only projections that
  rename each pair to one common alias before running benchstat. Count-one uses
  `CountOne/SQLite/KeyOnly`, `CountOne/SQLite/WatchChain`,
  `CountOne/PostgreSQL/KeyOnly`, and `CountOne/PostgreSQL/WatchChain`.
  Bulk comparison uses `Bulk32/SQLite/KeyOnly`,
  `Bulk32/SQLite/WatchChain`, `Bulk32/PostgreSQL/KeyOnly`, and
  `Bulk32/PostgreSQL/WatchChain`. Do not alter metric values. A bulk result
  fails when default benchstat reports `p < 0.05` and
  `C(Count32)/C(Sequential32) > 1.00`.
- For no-retry samples, key-only requires one transaction, at most `2N+3`
  write statements, and exactly `2N+3` affected rows. Watched requires three
  transactions, at most `2N+9` statements, and exactly `2N+9` rows. Thus count
  1/32/sequential use key-only `1/5/5`, `1/67/67`, `32/160/160` and watched
  `3/11/11`, `3/73/73`, `96/352/352`. Exclude BEGIN/COMMIT/read-only SQL;
  fewer statements require documented multi-row SQL, while extra transactions,
  statements, or rows fail immediately.

#### Primary Files

- `itest/list_on_test.go`
- `wallet/internal/db/itest/sql_role_cross_process_test.go`
- `wallet/fair_sqlite_sync_benchmark_test.go`
- `wallet/sql_role_allocation_baseline_benchmark_test.go`
- `wallet/sql_role_allocation_candidate_benchmark_test.go`
- `scripts/run-sql-role-benchmarks.sh`
- `scripts/run-itest-json-assert.sh`
- `scripts/assert-required-context.sh`
- `scripts/test-all-modules.sh`
- `scripts/run-required-static-checks.sh`
- `scripts/assert-ci-needs.sh`
- `Makefile`
- `make/testing_flags.mk`
- `.github/workflows/main.yml`

#### Exit Gate

- SQLite and PostgreSQL both complete the native btcd chain scenario with no
  `wallet.db` sidecar.
- KV preserves its legacy invariants and changes only for the shared behaviors
  listed in the backend capability matrix. SQL-only operations return the
  documented public unsupported result on KV.

## Implementation Order And PR Boundaries

The work should land as focused, independently testable changes. Do not defer
failure semantics to a final cleanup PR.

### PR 1: Freeze contracts and bind database identity

- Complete S0 evidence refresh; add wallet-owned account/status types, minimal
  public errors, 64-bit account/rescan selectors, checked conversions,
  recovery-window bounds, and external-package compile/high-ID tests.
- Update ADR 0005, normalize the remaining Go 1.25.11 workflow/docs settings,
  add the repository-wide toolchain drift check, and freeze
  `S0_SYNC_BASE_SHA` plus `SYNC_HARNESS_SHA`.
- Land the database-identity ADR, migration namespace, schema metadata, full
  genesis binding, PostgreSQL schema isolation, and incompatible preflight.
- Add canonical SQLite control identity, context-aware connection-bound
  migration executors, bootstrap locks, reconciliation, and crash tests.
- Use the bounded Manager-lifecycle migration adapter until PR 3 replaces it
  with public request-context transport.
- Freeze/checksum migrations 1-11, add upgrade fixtures, hold the Store-lifetime
  schema lease, and adopt the bounded SQLite pathname threat model.
- Add transaction-level PostgreSQL schema fencing, lease guard/PID/latch
  revocation, direct/session-pool configuration, affinity probe, and fail-closed
  PgBouncer transaction/statement rejection.
- Keep ordinary mutation intents, admission, runtime epochs, runtime fencing,
  and outbox behavior out of this PR. Use ordered commits so public-contract,
  toolchain/evidence, identity, and migration-executor review remain distinct.

### PR 2: Add modern chain side effects

- Land the typed context-aware exact watch registration and publication
  interfaces from the common contract with btcd/bitcoind/neutrino mappings.
- Add authoritative arbitrary-tx publication status/reorg proof with explicit
  unsupported behavior before ownership-free obligations can ship.
- Add bounded cancellation, mempool barriers, generation/idempotency checks,
  atomic staged chunks, stable subscription/delivery identity, unsupported
  disposition, exhaustive outcome tests, and prerequisite tagged releases.

### PR 3: Add the mutation and wallet-runtime foundation

- Land the quarantine/mutation-intent ADR and add health generations, intents,
  Store sessions, runtime epochs, and the wallet-wide outbox.
- Add PostgreSQL control/ownership connections, SQLite runtime locks,
  shared/exclusive admission, fencing, settled watermarks, opt-in policy
  registry, and reference-safe bounded intent/outbox/cursor/session GC.
- Add atomic snapshot/frontier acquisition, contiguous replay/cursors, runtime
  owner capabilities, account-watch state/registration identities, and
  transaction-boundary health/epoch checks. Land the coherent exact-script
  snapshot needed during starting-owner activation.
- Add protected-operation admission to Controller, signer, funding, key, and
  role-read entry points plus common reconciliation/failure seams.
- Land context-bearing Manager Create/Load and update every caller together with
  Controller queued/claimed/finished arbitration, SQL reconciliation contexts,
  and definitive-after-claim KV cancellation semantics.
- Add migration `000012_runtime_foundation`, server-derived PostgreSQL lock
  keys, graph admission, canonical-state schema, maintenance API/CLI/verifier
  registry, non-authoritative frontier, participants, and publication exemption.
- Own base create, existing-shape prepared sequential/imported account
  primitive, complete manifest-v2 request/effective-plan schema, guarded
  root-dependent normalization, one-time v1 partial-create adoption,
  initial-account intents, healthy-prefix resume/abort, completion, and
  Start/Stop capability matrix. Initially enable those policies plus owner
  transitions, birthday repair, and startup lease cleanup.

### PR 4: Correct vault, watch-only lifecycle, and import delivery

- Add `000013_script_protection`, authenticated descriptor envelopes, resumable
  post-Start unlock conversion, plaintext-watch activation, and encrypted-script
  open/sign blocking while legacy-v0 remains.
- Replace legacy script crypto selection with semantic protection and complete
  Manager-created SQLite/PostgreSQL restart conformance without regressing the
  current unlocked SQLite behavior.
- Define SQL watch-only authentication and stable KV unsupported behavior.
- Add non-destructive repeat-Unlock authentication and timer arbitration.
- Land reconciliation and outbox chain-registration delivery for changed raw
  address, script, tapscript, and secret mutations.
- Register script/import verifiers and implement the updated ADR 0005 contract.

### PR 5: Make live block application atomic

- Consume/extend PR 3's coherent snapshot for backend-specific canonical proof
  without changing legacy notification semantics.
- Apply connected-block transactions and contiguous tip together.
- Add migration `000014_canonical_chain`, database frontier arbitration,
  context-aware canonical proof adapters, prepared recovery horizons, graph
  admission, and operation verifiers.
- Add per-wallet attach proofs, durable unanimous participant agreement,
  database cutover-owner/takeover, non-authoritative migration-12 cutover, and
  targeted canonical ceilings.
- Own `RollbackToBlock`, wallet-scoped `RewindWallet`, database-wide
  `ResetChain`, coordinator restart order, policies, verifiers, reconciliation,
  and failure injection.
- Land sync-batch reconciliation and btcd/bitcoind/neutrino consistency tests.

### PR 6: Complete account identity, creation, and discovery

- Update ADR 0013 and add forward `000015_accounts_v2`; never rewrite migration
  5. Extend PR 3's prepared account primitive for exact/path-bound shapes and
  update SQL constraints/queries/verifiers plus every Go shape validator.
- Add exact/path-bound Store support, account intent/reconciliation, generated
  SQL, raw-v0-compatible KV fingerprint metadata, and unsupported adapters.
- Add the public account request, backend-capability/mode/XPub validation,
  fingerprint-preserving results, and guarded Manager normalization over the
  complete manifest-v2 fields.
- Own active watch-chain horizon registration/readiness and account verifiers.
  Land the immutable watched count-one baseline commit and record it as
  `ALLOCATION_BASE_SHA` before adding allocation behavior in later commits.
- Add `000016_discovery_allocation`, count-N allocation, invalid-child
  consumption, bounds, ordering, concurrency, intents, restart/reconciliation,
  prepared address CAS, and allocation/discovery verifiers.
- Add account-level watch-chain/key-only mode and `NewBulkAddresses` while
  preserving `NewAddress`/`GetUnusedAddress`; include outbox registration,
  repair, completion, and every literal candidate allocation benchmark.
- Do not expose exact-account or allocator behavior until the complete account,
  readiness, registration, and repair invariant is present at the PR tip.

### Dependency PR: Correct `wallet/txsizes`

- Fix post-change output CompactSize and per-input witness stack-count modeling.
- Prove exact/no-underestimate weight at 251/252/253 input/output and mixed
  boundaries, run isolated tests, and publish `wallet/txsizes/vX.Y.Z`.
- Before tagging, run `GOTOOLCHAIN=local GOWORK=off go test ./...` from
  `wallet/txsizes`.

### Dependency PR: Update `wallet/txauthor`

- Preserve exported v1 APIs and add V2 input/change/authoring APIs with checked
  totals, closed spendable change templates, and post-allocation validation plus
  aligned input/value/script/explicit spend-weight vectors.
  Support bounded generic P2WSH/Taproot script-path weight descriptors. Depend
  on the released txsizes tag, update callers/tests, and publish
  `wallet/txauthor/vX.Y.Z`.
- This PR is independently buildable and never uses a local `replace`.
- Before tagging, run `GOTOOLCHAIN=local GOWORK=off go test ./...` from
  `wallet/txauthor`.

### PR 7: Complete transactions, publication, PSBT, and funding

- Add full serialized-transaction identity checks and account-rename policy;
  remove unreachable SQL `DeleteTx` from the production Store contract.
- Expose marker-backed SQL branch invalidation with stable KV unsupported.
  Add `000017_terminal_invalidation`, exclusive graph admission, descendant
  rejection, and cancellation-safe rejection cleanup.
- Add `000018_publication_obligations`, ownership-independent full-byte
  publication/status proof, graph linearization, transitive terminal-parent
  disposition, retry/rebroadcast, and typed backend outcomes.
- Close remaining S0 ambiguity rows for labels, non-startup metadata/leases, and
  cleanup. PR 3 retains completion/birthday/startup cleanup; PR 5 retains
  rollback/rewind/reset.
- Consume both released nested modules, remove the local txsizes replacement,
  prove effective root txsizes/txauthor selection has no `Replace`, and retain
  the scoped wtxmgr replacement.
- Reject malformed/signed/finalized/SIGHASH_SINGLE or opaque-unsafe funding;
  preserve paired safe metadata through funding/BIP69 and freeze conflict-
  detecting `CombinePsbt` with no mutation on error.
- Add complete PSBT prevout/script/key/Taproot/sighash authorization, checked
  totals, every-input final script-engine verification, and SegWit parent rules.
- Route modern callers through txauthor V2, adapt sweepaccount/root input
  sources, and register non-dust change before returning any modern result.

### PR 8: Complete backend and chain acceptance

- Parameterize the complete native btcd scenario over SQLite and PostgreSQL.
- Add SQLite bitcoind/neutrino event and registration coverage.
- Add named required SQL-role CI jobs, stable selectors, and cross-process lock
  vectors.
- Use/validate the maintained benchmark runner, own workflow changes, add
  `itest-sql-role-cross-process-locks`, and run upgrade/maintenance/cancellation
  acceptance.
- Add one JSON-assert itest wrapper used by CI/final gates and attach named
  repository-administrator checked ruleset JSON evidence for required branch
  protection.
- Own `scripts/assert-required-context.sh`, `scripts/test-all-modules.sh`, the
  `run-required-static-checks.sh` manifest, `assert-ci-needs.sh`, and executable
  evidence artifacts.
- Run race, failure-injection, formatting, generation, lint, and reproducible
  benchmark evidence.

PR 2 depends on PR 1; PR 3 depends on PRs 1 and 2. The migration-bearing stack
is strictly linear: PR 4 depends on PR 3, PR 5 on PR 4, PR 6 on PR 5, and PR 7
on PR 6. txsizes tags before txauthor; PR 7 depends on both released modules.
PR 8 depends on every feature PR 1 through 7. Other required chain dependency
support tags before PR 2/PR 5. If PR 6 must split, its first part leaves exact
account/allocation behavior unreachable, does not redirect count-one, and cannot
create an unresolved watched intent. Never split an invariant so an intermediate
result can escape before reconciliation or chain registration exists.
The enumerated migration owner/version order is binding before freeze; an
out-of-order-ready PR waits rather than taking another version.
Each root PR may contain several ordered, independently reviewable commits; the
reduced PR count must not turn these eight root units into eight monolithic
commits.

## Verification Commands

Use a clean worktree for mutating check targets.

Focused checks during implementation:

```bash
make unit pkg=wallet case=TestName
make unit pkg=wallet/internal/db/... case=TestName
make unit-race pkg=wallet case=TestName
make itest-db db=sqlite
make itest-db db=postgres
```

Run fair-sync and allocation comparisons through one maintained harness:

```bash
set -euo pipefail
: "${EVIDENCE_DIR:?set an artifact directory outside the worktree}"
WORKTREE_ROOT="$(git rev-parse --show-toplevel)"
mkdir -p "${EVIDENCE_DIR}"
EVIDENCE_DIR="$(cd "${EVIDENCE_DIR}" && pwd -P)"
case "${EVIDENCE_DIR}/" in
  "${WORKTREE_ROOT}/"*) exit 1 ;;
esac
SYNC_BASE_SHA="$(git rev-parse '<S0 sync baseline>^{commit}')"
SYNC_HARNESS_SHA="$(git rev-parse '<sync harness>^{commit}')"
ALLOCATION_BASE_SHA="$(git rev-parse '<PR 6 baseline commit>^{commit}')"
CANDIDATE_SHA="$(git rev-parse 'HEAD^{commit}')"
./scripts/run-sql-role-benchmarks.sh \
  --sync-base="${SYNC_BASE_SHA}" \
  --sync-harness="${SYNC_HARNESS_SHA}" \
  --allocation-base="${ALLOCATION_BASE_SHA}" \
  --candidate="${CANDIDATE_SHA}" \
  --samples=10 \
  --output="${EVIDENCE_DIR}/sql-role-benchmark-artifacts"
```

The runner requires four immutable full SHAs, including `SYNC_HARNESS_SHA`.
It applies the identical benchmark-only harness to sync worktrees as specified
above. Allocation base must expose exactly the two baseline names; candidate
must expose all sixteen names, and the dedicated baseline-file SHA-256 must be
identical at allocation base/candidate. It rejects zero/missing/extra names,
baseline-body drift, production source drift from harness application, missing
artifacts, mutable refs, or allocation against the sync base. One linked
manifest records both distinct bases, harness/source hashes, discovered name
sets, commands, environments, raw samples, SQL writes, and benchstat output.
It creates isolated worktrees, alternates each base/candidate on one idle host,
and uses fresh databases. It provisions Go 1.25.11, exports
`GOTOOLCHAIN=local`, rejects any other
`go env GOVERSION`, pins btcd v0.25.0,
`golang.org/x/perf/cmd/benchstat@v0.0.0-20230227161431-f7320a6d63e8`, and the
PostgreSQL image by digest. It records `go version`, `go env GOVERSION`,
`GOTOOLCHAIN`, `GOMOD`, `GOWORK`, full SHAs, source hashes, tool/container
versions, PostgreSQL/pool settings, host OS/kernel/CPU, environment, commands,
raw samples, SQL write counts, and `benchstat` output. Automatic toolchain
download is not acceptance evidence.

Fair-sync compares only `S0_SYNC_BASE_SHA` to candidate; allocation compares
only `ALLOCATION_BASE_SHA` to candidate and uses sequential count-one for
count-N.
The runner exits nonzero on statistical, latency, allocation, or write-count
failure. Default benchstat confidence applies; `p < 0.05` plus duration or
allocation-per-block regression above 10 percent fails. Keep raw evidence; do
not make host-sensitive benchmarks generic hosted required checks without a
controlled runner.

The final gate assumes all implementation PRs, PostgreSQL harness support, and
maintained benchmarks have landed:

```bash
set -euo pipefail
export GOTOOLCHAIN=local
: "${EVIDENCE_DIR:?set an artifact directory outside the worktree}"
WORKTREE_ROOT="$(git rev-parse --show-toplevel)"
mkdir -p "${EVIDENCE_DIR}"
EVIDENCE_DIR="$(cd "${EVIDENCE_DIR}" && pwd -P)"
case "${EVIDENCE_DIR}/" in
  "${WORKTREE_ROOT}/"*) exit 1 ;;
esac
test "$(go env GOVERSION)" = "go1.25.11"
go version
make unit
make unit-race pkg=wallet
make itest-db db=sqlite
make itest-db db=postgres
make itest-db-race db=sqlite
make itest-db-race db=postgres
make itest-json-assert chain=btcd db=sqlite \
  expected='TestBtcWallet/btcd/wallet_sql_readiness'
make itest-json-assert chain=btcd db=postgres \
  expected='TestBtcWallet/btcd/wallet_sql_readiness'
make itest-json-assert chain=bitcoind db=sqlite \
  expected='TestBtcWallet/bitcoind/syncer_canonical_events'
make itest-json-assert chain=bitcoind db=sqlite \
  expected='TestBtcWallet/bitcoind/address_registration_repair'
make itest-json-assert chain=neutrino db=sqlite \
  expected='TestBtcWallet/neutrino/syncer_canonical_events'
make itest-json-assert chain=neutrino db=sqlite \
  expected='TestBtcWallet/neutrino/address_registration_repair'
make itest-sql-role-cross-process-locks
make lint-check
make fmt-check
make sql-format-check
make sql-lint-check
make sqlc-check
make tidy-module-check
./scripts/test-all-modules.sh
EVIDENCE_SHA="$(git rev-parse 'HEAD^{commit}')"
./scripts/assert-required-context.sh \
  --repo=btcsuite/btcwallet \
  --context='SQL role readiness' \
  --sha="${EVIDENCE_SHA}" \
  --output="${EVIDENCE_DIR}/required-context.json"
GOWORK=off go list -m -json \
  github.com/btcsuite/btcwallet/wallet/txsizes \
  >"${EVIDENCE_DIR}/txsizes-effective.json"
GOWORK=off go list -m -json \
  github.com/btcsuite/btcwallet/wallet/txauthor \
  >"${EVIDENCE_DIR}/txauthor-effective.json"
jq -e '.Version == "vX.Y.Z" and (.Replace | not)' \
  "${EVIDENCE_DIR}/txsizes-effective.json"
jq -e '.Version == "vX.Y.Z" and (.Replace | not)' \
  "${EVIDENCE_DIR}/txauthor-effective.json"
GOWORK=off go list -m -json \
  github.com/btcsuite/btcwallet/wallet/txsizes@vX.Y.Z \
  >"${EVIDENCE_DIR}/txsizes-release.json"
GOWORK=off go list -m -json \
  github.com/btcsuite/btcwallet/wallet/txauthor@vX.Y.Z \
  >"${EVIDENCE_DIR}/txauthor-release.json"
test -z "$(git status --porcelain --untracked-files=all)"
```

`test-all-modules.sh` discovers every tracked `go.mod`, sets
`GOTOOLCHAIN=local GOWORK=off`, and runs `go mod tidy -diff`, `go test ./...`,
and `go build ./...` from each module directory. It verifies fetched module
version/sum/directory against the reviewed release tags, rejects local replaces,
for txsizes/txauthor, explicitly permits the existing scoped wtxmgr replacement,
and snapshots cleanliness before/after so benchmark/test binaries, profiles,
coverage, or modified module files fail the gate. Effective root queries without
`@version` prove selected dependencies; separate release queries prove tag
provenance.

Also run:

- The complete native SQL wallet scenario on btcd for SQLite and PostgreSQL,
  then SQLite bitcoind/neutrino event and address-registration vectors.
- `git diff --check` after every PR.
- A final Go-comment pass over every touched handwritten Go file.
- The corrected fair sync benchmark before and after the sync change.
- The SQLite/PostgreSQL mutation-intent allocation benchmark before and after
  BW5.
- `GOTOOLCHAIN=local GOWORK=off go test ./...` in every nested module. The
  txsizes/txauthor root-module queries print released versions and no replace.

## Definition Of Done

This task is complete only when:

1. Every BWR issue is closed by code and the stated test evidence.
2. Schema family, migration ownership, and genesis identity are proven before
   any wallet mutation; canceled bootstrap returns a reconciled outcome and
   leaks no lock or connection.
3. Modern public roles expose no internal DB type or required internal error.
4. Recovery windows preserve values through `MaxRecoveryWindow`, reject larger
   values, and cannot overflow a child horizon.
5. Spendable and watch-only tapscripts survive SQL restart through the real
    vault; locked public roles return `ErrWalletLocked` without exposing an
    internal sentinel.
6. SQL watch-only lifecycle is explicit; KV preserves its documented
    unsupported-Unlock/idempotent-Lock behavior and remains locked.
7. A visible synced tip includes transaction, UTXO, and balance state through
   that block.
8. Exact custom and path-bound XPub accounts preserve truthful identity without
   filler accounts.
9. Bulk allocation is atomic, ordered, invalid-child safe, restart safe, and
    tied to the committing caller by its mutation intent; zero/max-plus-one fail
    under one public `MaxBulkAddresses` contract on every backend.
10. Key-only accounts remain absent from wallet ownership and chain watch
    material across ingestion, restart, and recovery.
11. No modern watched receiving or change script is returned before chain
    registration succeeds; chunked complete/additive delivery is atomic,
    successor-deduplicated, and key-only allocation performs no registration.
12. PSBT funding rejects signed/finalized/SIGHASH_SINGLE packets and
    unclassified opaque metadata, while preserving/reordering every admitted
    safe caller field.
13. Signing uses persisted wallet scope identity and validates complete
    prevout, script, key, effective default sighash, and Taproot authorization;
    finalization executes every extracted input before publishing the clone.
14. Public SQL branch invalidation is intent-backed, idempotent, and retains
    transaction history; KV reports unsupported without a persistence-format
    change.
15. Every Store mutation in the S0 inventory is reconciled before cache or
    side-effect publication; every owner transition has the closed control-plane
    reconciliation contract.
16. Quarantine behavior, maintenance authority, and unambiguous repair follow
    the accepted ADR and survive restart/concurrent Store access.
17. SQLite and PostgreSQL both pass the complete native btcd scenario without a
    KV sidecar.
18. The btcwallet entries in `wallet-default-sqlite-lnd-gaps.md` are removed or
    reassigned with concrete evidence; no stale issue remains open.
19. One active runtime epoch, wallet-wide contiguous outbox order, bounded
    completed event/cursor/session state, control-pool capacity, admission
    order, and crash fencing pass cross-process tests. Awaiting outage state is
    exempt until delivery.
20. The active default/master branch ruleset requires exact context
    `SQL role readiness` from the matching GitHub App integration; the named
    repository administrator's recorded `gh api` evidence proves it.
    Reproducible benchmarks meet every threshold.
21. PR 1 freezes migrations 1-11; each migration 12-18 freezes when its owning
    independently usable PR can apply it. Every forward upgrade fixture
    reaches a latest-equivalent schema while a Store-lifetime lease excludes
    old writers and old-version unresolved intent/quarantine is reconciled or
    returns maintenance-required before DDL.
22. Account, address, and recovery preparation performs no derivation, vault,
    encryption, or cache callback under a write transaction; CAS losers leave no
    durable effect.
23. Create/starting/active/cutover/maintenance capabilities are satisfiable and
    enforced, and database-wide canonical arbitration handles
    multi-wallet/process views without fencing dissent or vacuous all-fenced
    adoption.
24. Transaction-graph admission prevents descendants from racing invalidation;
    full-byte publication retry/status/rejection and transitive terminal-parent
    semantics survive cancellation, restart, reorg, and no-history replay.
25. Manager creation/loading and canonical chain proofs are context-aware and
    bounded; reset, shutdown, and Store close cannot hang on a backend request.
26. CI, every module, tools, and benchmark evidence execute Go 1.25.11 with
    `GOTOOLCHAIN=local`.
27. Benchmark evidence uses separate immutable `S0_SYNC_BASE_SHA` and
    `ALLOCATION_BASE_SHA`, identical recorded sync harness application, and
    exact distinct base/candidate benchmark-name sets.
28. Intent admission is enabled only for operation kinds with complete policy,
    verifier, reconciler, and failure seams; final coverage includes every
    reachable S0 writer.
29. Every account backend exposes durable `AccountWatchState`; pending or
    unsupported accounts cannot act as receiving authority.
30. Accepted-unconfirmed publications remain dependency-ordered rebroadcast
    obligations through restart until an authoritative terminal condition.
31. Every incomplete wallet-creation prefix has deterministic visibility,
    v1 adoption, resume, quarantine, cutover, and safe-abort behavior.
32. The typed mutation choke point and static write audit cover every SQL
    semantic writer, while repair releases all obsolete GC references.
33. Released txsizes/txauthor tags are independently fetchable at reviewed
    commits; txauthor v1 callers still compile, root txsizes/txauthor selection
    has no replacement, and every module tests/builds without leftover artifact.
34. S0 is pinned to `24e1343c9a3e5b6271efef200b1e277b1459dc0b` and records
    the complete six-prior-tip range-diff/rewrite classification before
    implementation.
35. CI aggregate result validation is event-aware, package/build-tag JSON
    assertions prove exact DB tests ran, and both evidence scripts execute in
    the final gate.

## Expected Review Shape

This is a size estimate, not a time estimate:

- Ten planned review units: eight root PRs plus the txsizes and txauthor
  dependency PRs. Expect 10-14 only when safe splits or other tagged chain
  dependency work are required.
- Approximately 120-200 handwritten production and test files materially
  changed.
- Approximately 16,000-26,000 handwritten production lines.
- Approximately 26,000-42,000 test lines.
- Additional generated sqlc churn from account identity, discovery policy,
  mutation intents, quarantine, runtime epochs/outbox, queries, and database
  identity changes.

The upper range is driven by the connection-bound migration executor, durable
wallet-runtime control plane, cross-process admission/ownership fencing,
backend parity, and operation-level failure injection. S0 must re-estimate after
its mutation and call-path inventories. If S0 proves a defect stale, remove its
work rather than spending the estimate.

Each PR may contain several small commits. Keep the repository's weighted
added-line guidance per commit; do not turn the size estimate into ten
monolithic commits.
