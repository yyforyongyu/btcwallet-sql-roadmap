# Task 302 — Round 5 review follow-ups (#1272, #1291, #1292)

Status: DONE through round 5c. Heads `7df7ca70c` (#1272) / `d7b17cb7a` (#1291) /
`d722e7b5c` (#1292); all three PRs 13/13 green, all twelve threads replied to and
resolved (paginated sweep reconciles 79/79, 114/114, 26/26 with zero
unresolved). Round-5a heads were `188516adf` / `08c89bea4` / `dfb34974f`, whose
#1272 jobs first failed on a Docker Hub outage
(`registry-1.docker.io` unreachable, so `bitcoind` extraction and the postgres
testcontainer both failed) and were re-run; the same commits pass in #1291 and
#1292.

Nine unresolved threads, five of them follow-ups on rev-3 answers rather than
new findings. Prior heads: `b1a44f8c8` / `68ab1f64e` / `77724271f`.

## Findings And Disposition

### F1 — signer godoc over-claims that no wallet mixes script secrecy

Raised twice (`gpt-5.6-sol`, and by the reviewer who authored the original
wording). **Accepted.** `Wallet.ImportTaprootScriptDeprecated` takes
`isSecretScript` from its caller and is part of the exported `Wallet` interface;
waddrmgr guards only the secret direction (`scoped_manager.go:2691-2699`
rejects secret-while-locked and secret-while-watch-only), while `ImportScript`
hardcodes `true`. A spendable kvdb wallet that P2SH-imports and taproot-imports
through the deprecated API therefore holds both key types at once, and the
repo's own benchmark exercises the non-secret half on a spendable wallet.

The claim is now scoped: the store path keys off the wallet-level bit on both
backends, and the legacy per-caller path is stated as the *reason* the reader
must read the flag instead of inferring it from the wallet. The contract godoc
on `AddressSecret.ScriptIsSecret` already hedged correctly and is unchanged.

### F2 — failure-delivery test barrier does not prove attachment

**Accepted.** `close(entered[i])` ran *before* `beginRun`, so a goroutine could
be descheduled past `run.release()`, attach to the succeeding generation, and be
admitted cleanly — the comment claimed a guarantee the code did not provide.

Attachment was observable from no existing state: a waiting run is registered
nowhere. `chainResetCoordinator` now carries a `waiting atomic.Int32`,
incremented under the mutex *after* the generation's completion channel is
captured, and the test waits for it to reach the waiter count before releasing
its run. Nothing in production reads it; the field documents why.

Falsified: consuming the error in `beginRun` (`active.err = nil`) fails the
test; restoring passes; 3× `-race` runs of the full package are clean.

### F3/F4 — behavior-affecting settings and the dial deadline

**Accepted, unified.** These arrived as separate threads (`statement_timeout`
plus pgx's exec mode and cache capacities; and `connect_timeout`) but have one
answer, and the reviewer is right that this branch introduces the shared pool,
so the inheritance is new behavior here rather than pre-existing.

They are not session *security*, so they do not belong in the security binding:
a mismatch means the two callers want differently shaped pools, not that one
may not connect — and putting them there would misreport a timeout difference
as a credential conflict. `runtimeStoreOptions` gains a `session` digest over
`ConnectTimeout`, `DefaultQueryExecMode`, both cache capacities, and the
`statement_timeout` / `lock_timeout` /
`idle_in_transaction_session_timeout` server parameters, so a difference is
refused as the ordinary options conflict alongside `maxConnections`. pgx
resolves its own defaults during `ParseConfig`, so an omitted setting and its
explicit default still share a pool. `application_name` stays cosmetic.

Note the lifetime this accepts: options bind for as long as a Manager knows the
identity, even across a clean close — the same rule `maxConnections` already
follows. Credentials stay generation-scoped so rotation still works.

### F5 — target_session_attrs case was vacuous

**Accepted.** The case added `&host=pg1,pg2` alongside the attribute, so
`fallbacksEqual` made it differ even with `pgValidateConnectID` collapsed to a
bool. The baseline row now varies only the attribute (proving validation
participates at all), and a dedicated test compares `read-write` against
`standby` over an identical route, asserting both validators are non-zero and
the fallback lists equal, so only the validator can carry the difference.

### F6 — commit message and PR description understate the change

**Accepted.** The commit hunk also makes `stopActiveWallets` the sole teardown
owner and join every `Stop` error; the message described only backend selection
and docs, and the PR body still said "Single commit" while the PR owns two.
Both updated.

### F7/F8 — 80-column violations in Makefile and ADR 0008

**Accepted.** The Makefile help line was 91 columns (103 before this branch
touched it) and is now split across two `#?` lines. The ADR bullet was 232
columns and is wrapped. The pre-existing "entire support matrix" coverage
promise, which this branch falsified by rejecting PostgreSQL, is now scoped to
the chain-backend matrix plus `sqlite`/`kvdb`. The rest of that document is
long-lined at base and was left alone.

### F9 — `behaviorParams` was a whitelist (round 5b)

**Accepted; the shape was wrong, not just the contents.** Listing the three
timeout GUCs silently shared every parameter it failed to mention —
`default_transaction_read_only`, `default_transaction_isolation`,
`synchronous_commit`, and hundreds more.

Inverted: `cosmeticParams` holds the only key a joining wallet may differ in
(`application_name`), and everything else pgx routes into `RuntimeParams` is
compared. That is exactly the server GUCs: pgx's `notRuntimeParams` consumes
every connection/TLS/auth setting, and `ParseConfigWithOptions` deletes its own
three keys. `search_path` / `options` remain excluded here only because session
security already compares them, so a mismatch is still reported as one.

Falsified by moving the four new keys into `cosmeticParams`: exactly those four
subtests fail.

### F10 — canonical contract still asserted wallet-level secrecy (round 5b)

**Accepted.** Round 5 corrected `signer.go` but left
`db.AddressSecret.ScriptIsSecret` claiming a spendable wallet's script is secret
and that kvdb yields the same wallet-level result — the very claim the signer
fix retracted. The contract now describes how each backend arrives at the value,
including that `ImportTaprootScriptDeprecated`'s caller-supplied flag lets one
spendable kvdb wallet hold both kinds. The locked-readability paragraph is
unchanged.

### F11 — session digest retained caller text (round 5c)

**Accepted.** `pgSessionSettings` returned raw `key=value` text and stored it in
`runtimeStoreOptions`, which binds for as long as a Manager knows the identity —
so a custom GUC carrying a credential stayed in registry memory after its pool
closed. That is the inverse of the treatment session security gets: that binding
holds a password in the clear because it must compare one, which is exactly why
it is generation-scoped and cleared on close. Options cannot be cleared, so the
value must not be there in the first place.

Now `pgSessionDigest`: each setting is rendered by `pgSetting` with both key and
value length-prefixed — the same injective framing `pgRouteIdentity` uses, so no
value can forge a delimiter and collide — then sorted, concatenated, and
SHA-256'd to hex. Conflicts already named no values, and the test now pins both
halves: the retained `options.session` after a clean close contains no secret,
and the conflict error names neither the secret nor the digest. Falsified by
returning the text instead, which leaks `myapp.token=16:hunter2-in-a-guc` into
the retained field.

### F12 — GUC names compared case-sensitively (round 5d)

**Accepted.** PostgreSQL compares GUC names ASCII-case-insensitively while pgx
stores whatever the DSN spelled, so `statement_timeout` and `Statement_Timeout`
were two entries: case-equivalent DSNs false-conflicted, and a differently cased
`Application_Name` or `Search_Path` fell into the wrong class entirely.

`pgRuntimeParams` now folds keys once (ASCII only, matching the server's
`guc_name_compare`) and both `pgSecurity` and `pgSessionDigest` consume that map
instead of `cfg.RuntimeParams`. A DSN setting one parameter twice with
*conflicting* values is refused, because the value the session keeps is whichever
pgx transmits last and map order leaves that unspecified; repeating one value is
unambiguous and stays allowed.

The extraction of `normalizePostgresTarget` is incidental — folding added a
branch that pushed `normalizeRuntimeStoreRequest` to cyclomatic 11 against a
budget of 10.

### F13 — CI never runs the SQLite harness path — NOT FIXED HERE

**Correct, and already fixed by the maintainer in the child PR.** All three
daemon itest jobs pin `db=kvdb` (`main.yml:241`, `:287`, `:356`), so this PR's
SQLite default is unexercised in CI. But #1300 (`GustavoStingelin`, based on
`wallet-default-sqlite`) adds `db: [kvdb, sqlite]` to all three jobs and deletes
the "kvdb only ... planned via a separate matrix" comment — commit `854cd4a79`,
`ci: run itest over sqlite and kvdb`.

This PR touches no workflow file. Adding a competing matrix here would duplicate
the maintainer's commit and conflict with the PR stacked directly on top. Locally
`make itest chain=btcd db=sqlite` passes, so the path itself is sound.

## Folding

| Finding | Commit |
| --- | --- |
| F1 | `wallet: route signer through Store` (#1272) |
| F2 | `wallet+db: coordinate shared chain resets` (#1291) |
| F3/F4/F5 | `wallet: share SQL runtime stores` (#1291) |
| F6/F7/F8 | `bwtest: default itests to sqlite, reject postgres` (#1292) |
| F9 | `wallet: share SQL runtime stores` (#1291) |
| F10 | `wallet+db: add address secret adapters` (#1272) |
| F11 | `wallet: share SQL runtime stores` (#1291) |
| F12 | `wallet: share SQL runtime stores` (#1291) |
| F13 | not fixed here — #1300 `854cd4a79` |

## Verification

Per-commit `go vet ./...` plus an itest-tagged vet; branch tests with `-race`;
four golangci passes at each of the three heads; an 80-column audit at tab
width 8 over every added line; `git diff --check`; GPG `%G? = G` on all commits.
