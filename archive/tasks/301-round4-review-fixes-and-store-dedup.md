# Task 301: Round-4 review fixes and store deduplication

Status: Rev 3 landed; awaiting review confirmation.

Rev 3 closed all ten reopened threads and is pushed (`b1a44f8c8` / `68ab1f64e` /
`77724271f`), all three PRs 13/13 CI-green, with every thread answered and
resolved after a paginated sweep (78/78, 114/114, 23/23). Status stays short of
Complete until a review pass confirms it, since rev 2 was marked complete and
then reopened.

Rev 3 also carries one commit that is not a review fix: `wallet: use the
default-account test constant`. CI's lint runs against a merge commit, where
golangci's new-from-rev attributes lines differently than a linear head, and it
surfaced a goconst finding on a `"default"` literal that predates this stack and
does not reproduce locally at the same SHA. The literals are now the package's
existing `defaultAccountName` constant, so the finding cannot fire under any
baseline. Lesson for the next round: re-lint every PR head *after* the final
fold and cascade -- linting the lower heads only is what let this reach CI.

What rev 3 changed:

- Worker liveness is an explicit `draining` flag rather than inferred from the
  request slots, which a live parked worker can leave both nil — that inference
  could spawn a second worker, clobber `applying`, and let `beginRun` admit a
  run and `waitIdle` return mid-rollback.
- Reset failures are delivered per generation: `applyErr` is gone, `beginRun`
  reads each generation's own `err` (so `req.err` has a reader again), every
  waiter of a failing generation learns, and the apply failure is logged where
  it happens. A run arriving after the drain sees no error by design; the
  documented recovery is the syncer re-detecting and re-queueing.
- `refusesJoin` excludes a closing generation, so a rotated credential arriving
  during the close window waits and is adopted instead of being refused.
- The security binding compares `require_auth` as a trimmed method set (it was
  a character multiset), identifies the `target_session_attrs` policy by
  validator code pointer (it was a bool, equating read-write with standby),
  adds both protocol-version bounds, and compares only `search_path`/`options`
  from `RuntimeParams` (comparing all of them made `application_name` a hard
  load failure). The doc claim is narrowed to what is enforced, since the
  `GetClientCertificate` rejection is a forward guard pgx never triggers.
- The `ResetChain` mock records heights, so boundary 1 and the full
  clamp-before-delete order are pinned; the signer godoc is rescoped by
  backend; the eight mock methods have comments.

Deferred, unchanged: pinning a parsed `ConnConfig` via `stdlib.OpenDB`.
`ConnectTimeout` is judged a pool option rather than session security, and is
tracked with that deferral rather than dropped.

Rev 2 folds in the GPT-5.6 review of rev 1 (blob `d3c55e03`) and four maintainer
threads that landed on #1272 after rev 1 was written. See Review Dispositions
for what was accepted, deferred, and rejected.

Target branches: `impl-secret-store` (#1272), `impl-runtime-store-6` (#1291),
`wallet-default-sqlite` (#1292).

Evidence tips: `4b94ace56` (#1272), `d7e0fb9f8` (#1291), `493c24f7e` (#1292).
All three are CI-green at these heads.

## Purpose

Close the eight unresolved review findings across #1272 and #1291, and apply
the two `wallet/internal/db/README.md` patterns that the changed store code
should be following but is not.

The findings and the refactors are one task because they touch the same commits:
RC-002, RC-003, RC-004, and RC-005 all rewrite the registry; RF-002 restructures
the same `ResetChain` body that RC-001's coordinator work sits behind; and
RF-001 lands in the same file as RC-006 and RC-007. Landing them separately
would rewrite `wallet: share SQL runtime stores` and `wallet+db: coordinate
shared chain resets` two or three times each.

## Discovery Gap

Four #1291 findings sat unresolved through two review rounds while the sweep
reported zero: `pullRequest.reviewThreads(first: 100)` truncates silently and
#1291 has 105 threads. A later sweep then reported zero on #1272 while it was
already at 77 threads with 4 unresolved, because new comments landed minutes
after the query.

The final sweep must therefore:

1. Paginate outer threads via `pageInfo.hasNextPage`, and paginate each thread's
   comments too.
2. Deduplicate node IDs and reconcile `totalCount` against unique IDs seen.
3. Re-run at the exact final head SHA of each branch, after the last push, not
   before.
4. Require every unresolved thread to have an answer, and assert zero
   unresolved.

This is a Definition of Done item, not a footnote.

## Unresolved Thread Register

Every unresolved thread with its disposition. IDs are GraphQL node IDs.

- `PRRT_kwDOALkxn86T4NkZ` — #1291, `reset_coordinator.go:212`:
  reset queued mid-apply is lost. → RC-001
- `PRRT_kwDOALkxn86T4JaY` — #1291, `runtime_store_registry.go:380`:
  security binding outlives a closed pool. → RC-002
- `PRRT_kwDOALkxn86T4JZ4` — #1291, `runtime_store_registry.go:423`:
  descriptor misses security-relevant fields. → RC-003
- `PRRT_kwDOALkxn86T4Ja_` — #1291, `runtime_store_test.go:935`:
  clean release is unasserted. → RC-004
- `PRRT_kwDOALkxn86UBO-J` — #1272, `data_types.go:632`:
  colocate `Validate`, shorten the receiver. → RC-006
- `PRRT_kwDOALkxn86UBPsT` — #1272,
  `pg/accountstore_getaccountsecret.go:102`:
  explain the row wrapper in the godoc. → RC-007
- `PRRT_kwDOALkxn86UBQ0_` — #1272,
  `sqlite/accountstore_getaccountsecret.go:103`:
  share `mapGetAccountSecretErr`. → RF-001
- `PRRT_kwDOALkxn86UBUD0` — #1272, `signer.go:957`:
  explain the key-type wrapper in the godoc. → RC-007

RC-005 has no thread of its own: it was found while assessing RC-003 and is a
defect this stack introduced.

## Issue Ledger

| ID | PR | Severity | Issue |
|---|---|---|---|
| RC-001 | #1291 | Critical | Reset queued mid-apply is dropped |
| RC-005 | #1291 | Critical | PG identity splits by user, and is not injective |
| RC-002 | #1291 | High | Security binding outlives the pool it protected |
| RC-003 | #1291 | High | PG security descriptor is incomplete |
| RC-004 | #1291 | Low | Clean lease release is asserted nowhere |
| RC-006 | #1272 | Low | `Validate` sits away from its type |
| RC-007 | #1272 | Low | Two wrappers lack a WHY godoc |
| RF-001 | #1272 | Medium | `GetAccountSecret` duplicates its conversion |
| RF-002 | #1291 | Medium | `ResetChain` duplicates a write invariant |

Severity is review impact. RC-001 and RC-005 are the only items that can change
or corrupt durable state; the rest are correctness-of-contract or hygiene.

## Review Dispositions

Accepted from the GPT-5.6 review: findings 1, 3, 4, 5, 6, 7, 9, 10 (inverted,
see RF-001), 11, 12, 13.

### Deferred: pinned PostgreSQL connection configuration

The registry parses the DSN to derive identity and security, but the pool is
opened from the raw DSN (`pg/store.go:47`, `sql.Open("pgx", cfg.Dsn)`). pgx
v5.10.0's `driverConnector.Connect` calls `pgx.ParseConfig(dc.name)` for **every
new physical connection** unless a config was pre-registered, so the exposure is
not a one-time parse race: passfiles, environment credentials, CA files, and
client certificates can change at any point in the pool's life, and later
connections will use the new material while `entry.security` still describes the
old.

The fix is to parse once and open the pool through the pinned parsed config
(`stdlib.OpenDB(*pgCfg)`), which changes the PostgreSQL opening contract in
`pg/config.go` and `pg/store.go` — pre-existing files this stack touches only
incidentally. That is out of scope here.

Therefore **RC-003 is explicitly limited to acquisition-time configuration
comparison**: it decides whether a second caller may join a live pool, based on
the configuration presented at acquire. It makes no claim about credential
material changing after the pool is open. Task 301 records file-backed
credential rotation and the pinned-`ConnConfig` opening contract as follow-up
work, to be raised against the PostgreSQL store rather than this stack.

### Rejected: marking a syncer unsynced at registration

The review proposed requiring `register` to mark a joining syncer unsynced
whenever a reset is pending or applying, on the grounds that it "may expose a
synced view". That state is unreachable:

- `syncStateBackendSyncing` is the `iota` zero value (`syncer.go:119`), so a
  freshly constructed syncer's `atomic.Uint32` already reads unsynced.
- `register` runs before the wallet is published, and a run cannot pass
  `beginRun` while pending or applying is set, so the syncer has no path to
  `syncStateSynced` during a reset.

Marking at registration is harmless defence-in-depth and may be added, but it is
not a defect and carries no test obligation.

## Scope Boundaries

Only methods these three branches change are in scope. Audited and excluded:

- `GetAddressSecret` — already shares its conversion through
  `db.AddressSecretRowToSecret`; residual twinning is query plumbing that must
  name sqlc types.
- `NewImportedAddress` — already has `db.NewImportedAddressOps` /
  `NewImportedAddressWithOps`, and #1272 changes only the shared file.
- `UpdateWallet`, `RollbackToBlock`, `CreateWallet`, `ListAccounts` — already
  ops-backed.
- `ListOutputsToWatch`, wallet-info conversion, `GetWallet`, `GetAddress`,
  `ResolveOwnedAddresses`, `ListLeasedOutputs` — thin reads.
  `ListLeasedOutputs` is named in the README's own exclusion list.
- kvdb lifecycle and adapter code — single backend, no twin to deduplicate.
- Pre-existing backend twins touched only incidentally (`runtime.go`,
  `store.go`, `walletstore_walletinfo.go`).

## RC-001: Reset Queued During Apply Is Lost

### Issue And Rationale

`execute` copies `full`, `height`, and `genesis` out of the pending request into
locals, releases the mutex, then calls `req.apply`. `queue` coalesces a later
request into that same struct (`wallet/reset_coordinator.go:212`) with no check
for whether the worker already snapshotted it.

A deeper rollback queued mid-apply therefore mutates a struct nobody reads
again. It never runs, its caller receives `errChainResetRestart` as though it
were accepted, and `pending.done` closes signalling a reset that did not include
it. A required rewind is silently skipped.

### Detailed Fix

1. Split the single `pending` slot into a coalescable `pending` and a
   non-coalescable `applying` request.
2. `queue` coalesces only into `pending`; when a worker is already applying,
   install a fresh `pending` rather than mutating the in-flight request.
3. Start a worker only when none runs, and make `execute` loop until `pending`
   is empty, so a request queued during an apply is applied next.
4. Snapshot the boundary under the mutex when promoting `pending` to `applying`,
   and never read the request's fields afterwards.
5. `beginRun` and `waitIdle` block while **either** slot is set, so no run is
   admitted between two coalesced generations.
6. Preserve coalescing precedence: a full reset supersedes pending rollbacks,
   otherwise the deepest height wins.

### Failure Semantics Across Generations

Rev 1 defined only the successful multi-generation path. Define explicitly:

1. A failed generation does not abort the drain: every queued generation is
   still applied.
2. Errors are retained across the drain — first error wins, or all are joined —
   and are not discarded by a later successful generation.
3. `beginRun` reports the retained error only once both slots are empty, so it
   never returns early while a generation is still queued, and never silently
   swallows an earlier failure.

### Tests

- A rollback queued mid-apply is applied afterwards at its own depth.
- A full reset queued during an apply still supersedes a pending rollback.
- Generation A fails while B is queued: B still applies, `beginRun` and
  `waitIdle` stay blocked through B, and A's error remains observable.
- `beginRun` admits no run between two applies.
- Participants stay unsynced from the first queue until the last apply.
- Existing quiescence and coalescing vectors stay green.

### Exit Gate

- No queued reset completes without its boundary applied.
- The worker drains every queued generation before exiting.
- A failure in one generation is neither lost nor fatal to the drain.

## RC-005: PostgreSQL Physical Identity

### Issue And Rationale

`runtime_store_registry.go:487` builds the identity as
`fmt.Sprintf("%s@%s:%d/%s", pgCfg.User, pgCfg.Host, pgCfg.Port,
pgCfg.Database)`.

Two defects:

1. **User is not part of a database's physical identity.** Two principals on one
   database get separate entries, pools, write gates, and reset coordinators, so
   one alias can write while the other performs a coordinated DB-wide rollback —
   exactly the hazard the canonical identity was introduced to prevent.
2. **The delimiter framing is not injective.** Verified: user `u`, host `/tmp`,
   port 5432, database `x:5432/d` and user `u`, host `/tmp:5432/x`, port 5432,
   database `d` both render `u@/tmp:5432/x:5432/d`. Reachable only with a
   database name containing `:` and `/`, so the risk is low, but the fix is two
   lines.

### Detailed Fix

1. Reduce the identity to the physical route and database only: host, port,
   database.
2. Frame it with explicit length prefixes so no delimiter can be forged.
3. Move `User` into the generation-scoped security binding of RC-002, so a live
   pool rejects a second principal while a closed pool may adopt one. This is
   fail-closed and also the correct answer for the case where two users would
   resolve different `search_path` schemas in one database: rejecting is safer
   than either splitting coordination or silently sharing a pool.

### Tests

- Two users, same database, pool live: conflict, not two pools.
- Two users, same database, after clean close: the second opens.
- The crafted collision tuple above yields two distinct identities.
- Equivalent URL and keyword spellings of one database still share a pool.

### Exit Gate

- One physical database has exactly one entry, pool, write gate, and reset
  coordinator regardless of principal.
- No two distinct routes can produce one identity.

## RC-002: Security Binding Outlives Its Pool

### Issue And Rationale

Entries are never removed from `r.stores`, and a clean final close preserves
`entry.options` to keep the Manager-scoped pool-limit and network binding.
`security` was added to that struct, and the `entry.options != req.options`
comparison sits above the state switch, so it is reached when
`entry.store == nil`.

After the last wallet on a PostgreSQL database stops, re-opening it with a
rotated password is rejected for the rest of the Manager's lifetime, reporting
that the store "is already open with different connection credentials" while
nothing is open. Rotation between a stop and a later load is ordinary
operations work.

Unlike `network`, which binds a database to one chain permanently, the security
binding only needs to hold while a pool is live or opening: nothing can be
inherited from a closed pool.

### Required State Table

Rev 1 defined only live and opening. Define all six:

| Entry state | Behaviour on mismatched security |
|---|---|
| Fresh (no generation) | Adopt the request's binding |
| Opening | Reject: the in-flight pool would be inherited |
| Live (`store != nil`) | Reject: the live pool would be inherited |
| Failed open, draining | Reject while reservations remain |
| Closing | Wait for the close, then loop and adopt |
| Clean-closed | Adopt the request's binding |
| Close failed (quarantined) | Return the quarantine error, ignoring security |

### Detailed Fix

1. Move `security` out of the compared `runtimeStoreOptions` into its own entry
   field with the lifetime above.
2. Set it when a generation starts opening; clear it on clean final close
   alongside `store`, `closeFn`, and `reset`.
3. Compare it **after** the `closing` branch, so a closing entry waits and then
   adopts rather than rejecting immediately.
4. Clear it once the last failed-open reservation drains, so corrected
   credentials are not blocked.
5. Keep `maxConnections` and `network` comparisons unchanged.

### Tests

- Mismatched security while live, opening, and failed-open draining: conflict.
- Mismatched security arriving during a clean close: waits, then opens.
- Mismatched security after the last lease released: opens.
- Corrected credentials after a failed open: opens.
- A close failure still quarantines regardless of requested security.

### Exit Gate

- A credential or TLS rotation across a stop/load cycle is not an error.
- No concurrent alias can inherit a live or opening pool's session security.

## RC-003: PostgreSQL Security Descriptor Is Incomplete

### Issue And Rationale

The descriptor renders three TLS fields, so differences it exists to catch hash
identically and a second alias runs under the first caller's session security.
Verified against pgx v5.10.0, all of these collide on one database:

- `sslmode=require` vs `sslmode=verify-ca`: with no `sslrootcert` both set
  `InsecureSkipVerify` and leave `RootCAs` nil; only `VerifyPeerCertificate`
  differs and it is not rendered, so a caller asking for chain verification
  silently joins a pool doing none.
- `require_auth` and `channel_binding` are never read, so libpq's downgrade
  protection can be neutralized by whichever alias opens first.
- Distinct `sslrootcert` files both render `roots=true`.
- `sslcert`/`sslkey` populate `TLSConfig.Certificates`, never rendered.
- `sslnegotiation` is unread.

`RootCAs.Subjects()` is also a weak identity: it is deprecated, carries only
subjects, so distinct CA keys sharing a subject collide.

The `ssl*` keys cannot be recovered from the parsed config — `pgconn` strips
them from `RuntimeParams` — so the binding must be built from structured fields.

### Scope: Acquisition-Time Only

RC-003 governs whether a caller may join a live pool based on the configuration
presented at acquire. It makes no claim about material changing after the pool
opens; see Review Dispositions for why that is follow-up work.

### Decision: Structured Binding, Not A Connection-String Hash

Hashing `cfg.ConnString()` is fail-closed against fields pgx adds later, but it
returns the caller's original text, so URL and keyword spellings of one database
would conflict — and an earlier finding on this same line required equivalent
spellings to share a pool, with a test pinning it. Build a structured binding
with an explicit equality method instead, and get the fail-closed property from
rejecting configuration this code does not model.

### Detailed Fix

1. Replace the digest with a structured binding type carrying an `Equal` method.
2. Compare root pools with `x509.CertPool.Equal`, not `Subjects()`.
3. Digest actual client-certificate DER, and include `NextProtos` and
   `VerifyPeerCertificate != nil`.
4. Include every `Fallbacks` endpoint (host and port, not only TLS), Kerberos
   service and SPN, TLS protocol limits, `RuntimeParams` (notably `options` and
   `search_path`), and `ValidateConnect` presence.
5. Normalize to canonical effective values: an omitted `sslnegotiation` equals
   explicit `postgres`, and `require_auth` allow-set order must not matter, so
   sort maps and allow-sets.
6. **Reject** DSNs carrying security- or session-affecting parameters this code
   does not model, rather than silently omitting them. This is what makes the
   binding fail-closed as pgx grows.
7. Keep bindings out of error text: the conflict message names only
   "credentials or TLS settings".

### Tests

- `require` vs `verify-ca` conflicts.
- `require_auth=scram-sha-256` vs `require_auth=password` conflicts.
- Distinct `sslrootcert` files conflict; two CAs sharing a subject conflict.
- Reordered equivalent roots and reordered allow-sets do **not** conflict.
- Presenting a client certificate vs not conflicts.
- Differing fallback routes, Kerberos SPN, protocol limits, `search_path`, and
  `target_session_attrs` conflict.
- An unmodeled parameter is rejected, not ignored.
- Equivalent URL and keyword spellings still share one pool.
- No error text contains a password or raw DSN.

### Exit Gate

- Every field above participates in the comparison or is rejected.
- Equivalent spellings still share one pool and reset coordinator.

## RC-004: Clean Lease Release Is Unasserted

### Issue And Rationale

`TestManagerCreateRuntimeWalletLeaseSucceeds` discards the release function, so
the clean release path it claims to verify is asserted nowhere; only the failure
path is. Asserting `require.NoError` alone would also pass against a no-op
release that never closed the pool.

### Detailed Fix

Capture the release function; count closes in the injected close function.
Assert the first release returns nil and closes exactly once, and that a second
release is idempotent and does not increment the count.

### Exit Gate

The test fails if release stops closing the final pool.

## RC-006: Query Validation Placement

### Issue And Rationale

`GetAccountSecretQuery.Validate` lives in
`wallet/internal/db/accountstore_getaccountsecret.go` while the type it belongs
to is declared in `data_types.go:632`, and the receiver is spelled `query` where
the package convention is a short receiver.

### Detailed Fix

Move `Validate` next to its type and shorten the receiver to `q`. This also
leaves `accountstore_getaccountsecret.go` holding exactly the shared row and
converter RF-001 adds.

## RC-007: Missing WHY Godoc On Two Wrappers

### Issue And Rationale

Two wrappers document what they are but not why they exist:

- the account-secret column wrapper
  (`{pg,sqlite}/accountstore_getaccountsecret.go:102`), whose reason is that a
  generic type parameter cannot name fields common to a constraint union, so
  the row type must be resolved once before conversion;
- the script crypto-key selector at `signer.go:957`, whose reason is that the
  store reports which key the ciphertext is under and the wrapper maps that
  secrecy onto the vault's key type so the read decrypts with the storage key.

### Detailed Fix

State the reason in each godoc. The column-wrapper explanation lands on the
shared `db.AccountSecretRow` introduced by RF-001, not on a per-backend copy.

## RF-001: Share The Account-Secret Conversion

### Issue And Rationale

`wallet/internal/db/{pg,sqlite}/accountstore_getaccountsecret.go` are 235-line
files that are byte-identical apart from the package clause and the sqlc import
path. Every sibling entity already shares its conversion:

| Entity | Shared row | Shared converter |
|---|---|---|
| Account info | `db.AccountInfoRow[int64]` | `db.AccountRowToInfo` |
| Account props | `db.AccountPropsRow[int64]` | `db.AccountPropsRowToInfo` |
| Address secret | `db.AddressSecretRow` | `db.AddressSecretRowToSecret` |
| Account secret | — | — |

Account secrets are the newest and the only holdout. The cost is already paid:
the identical fix had to be applied to both copies three times in two review
rounds (a missing type-switch `default`, a complexity extraction, and an
82-column wrap).

No `ops` interface: the method runs one selected query and converts one row,
with no ordering invariant for a shared helper to own. The README's thin-read
exception is not applied consistently in this package — `GetTxDetailOps` and
`ListTxDetailsOps` exist despite `GetTx` and `ListTxns` being named there — so
the governing argument is the sibling precedent above, not that paragraph.

### `mapGetAccountSecretErr` Moves To Shared

The GPT review read README:47-48 ("query error mapping stay in backend files")
as forbidding this, and rev 1 left it optional. Both are wrong here, and the
maintainer thread asking for the move is right:

- The function imports only `context`, `database/sql`, `errors`, `fmt`, and
  `db`. It names **no** sqlc or driver type.
- Its only backend coupling is `sql.ErrNoRows`, which is `database/sql`, shared
  by both backends.
- It is byte-identical in both files.
- The README clause targets driver-specific mapping, which lives in each
  backend's `ClassifyError`/`mapErr`. This function is selector-shaped domain
  error construction, not driver mapping.

Move it to a shared `wallet/internal/db/accountstore_common.go`, the sibling of
the existing `db/addressstore_common.go`. Neither backend currently has a local
`accountstore_common.go`, so no backend file is displaced.

### Detailed Fix

1. Add `db.AccountSecretRow` mirroring `db.AddressSecretRow`, carrying the WHY
   godoc from RC-007.
2. Add `db.AccountSecretRowToSecret` with the wallet-ID, scope, account-number,
   and fingerprint conversions and null handling, preserving the existing
   per-field error contexts.
3. Move `mapGetAccountSecretErr` to `db/accountstore_common.go`.
4. Reduce each backend to what names sqlc types: the two `execRead` calls, the
   row-type constraint, and the type switch filling `db.AccountSecretRow`.
5. Keep the type-switch `default` guard in each backend — it is what makes a
   future constraint member fail loudly instead of yielding a zero-valued
   secret.

### What Must Not Move

The `accountSecretRow` constraint and the type switch name generated sqlc row
types and stay backend-side. About 72 lines per backend move, plus the error
mapper.

### Tests

- Shared unit tests in `db/accountstore_getaccountsecret_test.go` covering
  overflowing wallet ID, purpose, coin, account number, and fingerprint, and
  absent versus present-zero fingerprint.
- Shared unit tests for the error mapper across all three selectors and the
  non-`ErrNoRows` passthrough.
- Existing sqlite and pg conformance vectors stay green with no behaviour
  change.

### Exit Gate

- One copy of the conversion and one of the error mapping.
- Neither backend file retains numeric conversion logic.
- No observable behaviour or error text changes.

## RF-002: Share The ResetChain Write Invariant

### Issue And Rationale

`ResetChain` is byte-identical in both backends
(`wallet/internal/db/{pg,sqlite}/txstore_rollback.go`) and encodes an ordered
invariant inside one transaction: ensure the genesis row exists, then run the
DB-wide rollback from height one. `RollbackToBlockOps` already exists and
`ensureBlockExists` is genuinely backend-local (`{pg,sqlite}/block.go`, taking
`*sqlc.Queries`), so this is the README's ops shape — a multi-stage write whose
sequencing should live in one place. The README already lists the sibling
`RollbackToBlock` as an ops user.

### Detailed Fix

1. Add `db.ResetChainOps`, embedding `db.RollbackToBlockOps` and adding
   `EnsureBlock(ctx, *db.Block) error`.
2. Add `db.ResetChainWithOps` owning the transaction-local sequence, preserving
   the existing error contexts.
3. Extend each backend's `rollbackToBlockOps` adapter with `EnsureBlock`
   delegating to its local `ensureBlockExists`.
4. Document the ordered stages on both the interface and the shared helper.

### What Must Not Move

Genesis validation and the write gate stay in the backend methods.
`db.ValidateGenesisBlock` currently runs before the gate is acquired; moving it
into the shared helper would run it inside the gate and the transaction and lose
that fail-fast. The gate is backend state, as is transaction ownership.

### Tests

- Fake-ops vector for successful stage ordering.
- Fake-ops vector proving an `EnsureBlock` failure short-circuits before
  rollback.
- Fake-ops vector proving a rollback failure propagates.
- The `ensure genesis block` error context is unchanged.
- Existing sqlite and pg `ResetChain` vectors stay green.
- Invalid genesis still fails before the write gate is acquired, on both
  backends.

### Exit Gate

- One copy of the reset sequence.
- Write-gate and transaction ownership stay in the backends.
- Invalid genesis still fails without acquiring the gate.

## Commit And PR Boundaries

Everything folds into the commit that introduced the code. No standalone
review-fix commits.

| Work | Fold target | PR |
|---|---|---|
| RF-001, RC-006, RC-007 row | `wallet+db: add account secret adapters` | 1272 |
| RC-007 key-type | `wallet: route signer through Store` | 1272 |
| RC-001, RF-002 | `wallet+db: coordinate shared chain resets` | 1291 |
| RC-002/3/4/5 | `wallet: share SQL runtime stores` | 1291 |

RF-001 rewrites #1272, so #1291 and #1292 must be cascade-rebased even though
#1292 carries no change. Order: #1272, then #1291, then #1292.

## Cascade And Push Procedure

Folding into old commits rewrites every downstream commit, so all three pushes
are non-fast-forward and the two remotes cannot be updated atomically. A
mis-sequenced push already cost a branch once in this stack's history.

1. Create backup refs at all three evidence tips before starting.
2. Complete and verify the whole cascade locally before pushing anything.
3. Re-read both remotes, then push each branch with an explicit
   `--force-with-lease=refs/heads/<branch>:<old-tip>`.
4. Never chain a piped `git rebase` into `&&`: the pipeline reports the
   pipe's status, so a mid-conflict `git branch -f` can clobber the branch.
5. After each push, verify both remote refs, the GitHub `headRefOid`, and the
   downstream PR's `baseRefOid` before advancing to the next branch.

## Verification Commands

Run mutating check targets in a clean worktree only, asserting a clean status
before and after. Pin the toolchain to the `go` directive in `go.mod`.

```bash
GOWORK=off go test ./wallet/... ./bwtest/... -count=1
GOWORK=off go test -race ./wallet/ \
    ./wallet/internal/db/{sqlite,pg,kvdb,runtime}/
make itest-db db=sqlite nocache=1
make itest-db db=postgres nocache=1
make fmt-check
make sqlc-check
make lint-check
git diff --check
```

The account-secret and `ResetChain` conformance files are `//go:build itest`, so
`go test ./wallet/...` does not run them; the `itest-db` targets above are
required evidence, not optional.

Per-commit and per-head checks:

- `go vet ./...` at every commit in the stack, in a disposable clean worktree.
- Four golangci passes at each of the three PR heads (default, `./itest`, and
  the sqlite and postgres DB-itest tag sets).
- `git verify-commit` across each immediate PR-base range, asserting the
  expected signing fingerprint on every rewritten commit.
- An executable 80-column audit at tab width 8 over the added lines between old
  and new tips — golangci's `lll` runs at `tab-width: 4` and cannot see that
  class.

## Definition Of Done

1. RC-001 through RC-007 are closed by code plus the stated tests.
2. `GetAccountSecret` has one shared conversion and one shared error mapper;
   `ResetChain` has one shared sequence.
3. One physical PostgreSQL database maps to one entry, pool, gate, and reset
   coordinator regardless of principal.
4. No excluded method from Scope Boundaries was touched.
5. Every commit is GPG-signed with the expected fingerprint and compiles
   standalone.
6. All four lint passes are clean at all three heads, and no added line exceeds
   80 columns at tab width 8.
7. The `itest-db` targets pass for both backends at the rewritten heads.
8. The three PRs are green, and a paginated sweep at the exact final head SHAs
   reconciles `totalCount` against unique thread IDs and reports zero
   unresolved.
9. The deferred pinned-`ConnConfig` follow-up is filed against the PostgreSQL
   store, not left implicit here.

## Expected Review Shape

- 9 folded changes across 4 commits in 2 PRs.
- Roughly 400 production lines changed; RF-001 and RF-002 are net reductions,
  RC-003's structured binding is the main addition.
- Roughly 450 test lines added, dominated by RC-001's generation-failure
  vectors and RC-003's comparison matrix.
