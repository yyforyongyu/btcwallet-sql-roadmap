# Task 340 — bind SQL databases to schema and network identity

Component: **Manager and SQL Store**.

Dependencies: **An accepted [`340.identity-revalidation`][t340] record; no task
dependencies**.

## Rationale

Migration version alone does not prove a database belongs to the btcwallet role
schema or the intended Bitcoin network. A foreign or wrong-network database
must fail before ordinary wallet queries mutate it.

## Pre-code gate

[`340.identity-revalidation`][t340] records an accepted review of the identity tuple,
custom-signet input, read-only preflight and empty-database adoption order, and
database-native first-open exclusion against the current SQL Store branches.
It must confirm that the proposed behavior digest is minimal and sufficient and
that [Task 1000][t1000]'s lock, manifest, and generic recovery machinery remain outside
scope. This is a [Task 340][t340] design artifact, not a separate implementation task;
implementation starts only after it is accepted.

## Details

- Persist a schema-family identifier, generation/version, genesis hash, network
  magic, and a canonical digest of the parameters that change maintained Wallet
  behavior: a canonical network-family/name discriminator, coinbase maturity,
  address prefixes and Bech32 HRP, HD private and public versions, and BIP44
  coin type. For custom signet, include the full canonical challenge hash; the
  truncated magic alone is not unique.
- Carry the canonical custom-signet challenge digest explicitly in the unified
  `ManagerConfig` and Store identity input. Do not attempt to recover it from
  `chaincfg.Params`, which retains only the truncated network magic.
- Deep-copy `ManagerConfig.ChainParams` and the explicit signet challenge input
  before identity preflight. Retain and use only Manager-owned immutable
  snapshots, including copies of nested mutable data used by Wallet behavior.
  Wallet configuration and information APIs return copies rather than exposing
  the retained parameter pointer or mutable challenge bytes.
- Require the behavior digest, and the challenge digest for signet, to be
  present and match before database access. The signet challenge is not
  optional for default or custom signet merely because the truncated network
  magic is available.
- Keep btcwallet's PostgreSQL schema and migration namespace explicit.
- Probe identity before ordinary migrations or wallet queries can mutate the
  database.
- Adopt only an empty unmarked database. Reject a populated unmarked database
  unless existing persisted facts prove its identity without mutation.
- Reject dirty, newer, foreign, and mismatched databases before wallet access.
- Preserve immutable forward migrations after the first marked release.
- Follow the accepted [`340.identity-revalidation`][t340]; do not copy [Task 1000][t1000]'s
  large lock, manifest, or generic recovery design.

## Potential fix

Add a read-only preflight followed by a minimal metadata bootstrap transaction
protected by the database's native migration exclusion. Reopen verifies exact
family, version, genesis, and behavior digest before running forward migrations
or constructing a Store. Build that identity from one deep-copied Manager-owned
network snapshot and never expose the snapshot directly. Add the required-for-
signet challenge digest directly to the existing unified configuration rather
than introducing a backend-specific config type.

## Done when

SQLite and PostgreSQL tests cover empty initialization, matching reopen,
wrong-network and different-custom-signet reopen, foreign schema, dirty/newer
versions, and concurrent first open without mutating rejected databases. A
same-genesis fixture that changes each digested behavioral field is rejected,
as is a fixture with colliding truncated signet magic but a different full
challenge hash. An otherwise identical renamed signet is rejected. Mutating the
caller's original parameters or challenge after `NewManager`, and mutating any
network information returned by a Wallet, cannot change Manager behavior or its
next identity check. Missing and mismatched behavior or required signet digests
fail before database access on both dialects.

[t340]: 340-durable-database-identity.md
[t1000]: ../archive/tasks/1000-complete-btcwallet-sql-role-readiness.md
