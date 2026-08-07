# Task 335 — convert existing kvdb wallets to SQL

Component: **Manager migration and WalletVault**.

Dependencies: **[Task 332][t332], [Task 337][t337], [Task 340][t340], and
[Task 345][t345]**. It does not depend on modern-kvdb derivation in
[Task 361](361-kvdb-imported-xpub-derivation.md) or live-watch delivery in
[Task 364](364-durable-watch-registration.md).

## Rationale

kvdb remains the legacy dual-encryption backend. Compatibility needs one
conversion to SQL, not a new intermediate kvdb format. Removing the modern
kvdb backend is the later [Task 353][t353].

## Details

- Never open the source through an upgrade-capable path. Decode it read-only or
  operate on a consistent immutable copy while verifying the source is
  unchanged.
- Require an explicit trusted source network and, for signet, its trusted full
  challenge digest. Treat that input as authoritative because legacy kvdb does
  not persist enough information to prove which equal-magic signet challenge
  was used. Validate persisted facts that encode a network against the trusted
  input before writing [Task 340][t340] identity metadata; do not infer the network by
  passing parameters to `waddrmgr.Open` or claim the legacy file authenticates
  the challenge.
- Decrypt each script with its persisted per-record cipher provenance.
- Bound every plaintext root, account key, private key, and script lifetime and
  zero each working buffer on success, refusal, cancellation, and write failure.
- Decode extended keys inside a wipeable migration boundary. If an upstream API
  forces an immutable plaintext string, retain no reference after parsing and
  document and test the same bounded exposure accepted by [Task 329][t329].
- Re-encrypt secrets under the SQL wallet's single-passphrase Vault model.
- Import full transaction history, UTXOs, and sync state. Do not make an
  incomplete rescan destination appear atomically ready.
- Map kvdb's synthetic imported account and scoped buckets into accountless SQL
  raw addresses as required by ADR 0013.
- Map true imported-XPub accounts to SQL accounts with no BIP44 account number;
  retain their scope, XPub, fingerprint optionality, branch counters, and child
  paths without copying the kvdb synthetic account number.
- Persist `watch-chain` for every migrated legacy account and accountless raw
  import. Legacy kvdb has no key-only mode, so migration must not infer one.
- Convert every ordinary derived account and scope, including its name, schema,
  encrypted account material, external and internal counters, and persisted
  child paths. Do not infer missing accounts from only address rows.
- Normalize each SQL `next_account_number` from the legacy allocation state and
  the greatest ordinary numbered account using checked arithmetic. Synthetic
  imported-account numbers and numberless imported XPub accounts must not move
  that cursor. Refuse inconsistent or overflowing source state.
- Refuse any scope whose legacy accounts have conflicting address-schema
  overrides, unless account-level schema storage is separately added to SQL.
- Reject a spendable kvdb wallet containing grandfathered watch-only material
  with a typed, non-mutating error. Supporting a split into spendable and
  watch-only SQL wallets requires a separately approved migration extension.
- Reject root-neutered or other legacy custody shapes that the SQL watch-only
  invariant cannot represent, unless the caller supplies an explicitly approved
  replacement root transformation.
- Transform the complete transaction graph: spent credits, debit edges, change
  provenance, receive times, block order, labels, status, UTXOs, and replacement
  relationships. Map kvdb pending and published side-bucket membership to SQL
  status; absence from the pending bucket means published.
- During immutable preflight, decode every confirmed and unmined transaction and
  require its computed txid to equal the independent legacy record key. Refuse
  any mismatch with a graph-corruption reason before destination mutation.
- Validate the independently serialized monetary graph during the same
  preflight. Every transaction output and checked output aggregate must be in
  Bitcoin's monetary range. Every mined or unmined credit index must exist in
  its creator transaction, and its serialized amount must equal that output.
  Every mined debit index must exist in its spender transaction, whose input
  outpoint must name the referenced credit; the serialized debit amount must
  equal that credit and creator output.
- Require mined spent-credit pointers and debit records to resolve to each
  other exactly. Refuse missing, one-sided, mismatched, or out-of-range credit
  and debit records as graph corruption before destination mutation.
- Reconstruct the canonical transaction graph independently and require exact
  bidirectional equality with the legacy mined-balance value, block-membership
  indexes, unspent-outpoint index, and unmined-input spender index. Refuse every
  missing, extra, malformed, or misdirected entry before destination mutation;
  none of these secondary structures is authoritative over the decoded graph.
- Refuse a legacy outpoint with multiple unmined spenders before mutation. The
  SQL UTXO model has one spender pointer, so selecting one edge would lose
  history and is not a safe normalization.
- Preflight duplicate raw script identities stored in different legacy scopes.
  Merge only when every decrypted script, provenance, usedness, and public fact
  is semantically identical after ADR-0013 account removal; otherwise return a
  dedicated duplicate-identity refusal before mutation.
- Preflight wallet-wide script collisions across raw and derived records. Only
  semantically identical all-raw duplicates may merge. Any raw-versus-derived
  or derived-versus-derived collision receives the duplicate-identity refusal;
  migration must not discard or change an owner to satisfy SQL uniqueness.
- Compare normalized account-XPub payloads wallet-wide even when no address has
  yet been derived. If two accounts have equal payloads and overlapping
  effective external or internal branch schemas, refuse the latent script
  collision before destination mutation, including across scopes.
- Normalize legacy change flags from the proven internal branch. Refuse the
  source before mutation if a persisted flag conflicts with its derivation and
  the SQL model cannot retain that discrepancy truthfully.
- Reject legacy duplicate-txid history before mutation unless a deterministic
  BIP30-safe normalization is separately specified.
- Import active unexpired lease IDs and absolute expirations, or refuse to start
  while any active lease cannot be represented exactly.
- Prove bidirectional usedness equivalence for every address: its sticky legacy
  bit must equal whether the SQL destination contains a retained `utxos` credit
  row, the exact ADR-0011 source. Refuse both true-without-credit and
  false-with-credit shapes before destination mutation.
- Cover spendable and watch-only wallets.
- Import into a fresh destination in one migration-specific SQL transaction;
  ordinary independently committing Store methods are not an atomic importer.
- Verify the destination before any caller-selected cutover. Keep the source
  bbolt file as the rollback artifact.
- Require trusted normalized wallet metadata, including destination name and
  creation or birthday facts, from the migration request. Do not treat the kvdb
  adapter's caller-supplied name or defaults as persisted source provenance.
- Treat an aggregate import commit error as indeterminate. Reopen and reconcile
  the exact destination identity and graph before reporting ready or failed.
- Return one wallet-owned migration-refusal sentinel with a reason code for
  network, cursor, schema, custody, multi-spender, duplicate identity, graph
  corruption, change, duplicate-txid, lease, and usedness shapes. Every reason
  is asserted with `errors.Is` plus its typed detail and leaves the destination
  unmodified.

## Suggested fix

With the source wallet closed, take a consistent copy and run any required
legacy upgrades only on that copy. Decode the copy and write a fresh SQL
database through a backend-local aggregate import transaction. Verify identity,
balances, scripts, history, and sync state before reporting it ready. Do not add
a generic import manifest or widen normal Store methods into a migration API.
SQLite is the required first destination; PostgreSQL conversion remains outside
this task until its Manager path is scheduled. Make normalized wallet metadata
an explicit migration input and validate it before opening the destination.

## Verification

A representative real kvdb wallet round-trips to SQL with equivalent identity,
balances, scripts, history, and sync state; injected import failure leaves no
partial destination; raw imports gain no fake account identity; mixed-mode
wallets and unsupported custody or duplicate-txid shapes fail without mutation;
ordinary and imported accounts, active leases, status, change provenance, and
the full transaction graph round-trip; scope cursors are normalized without
synthetic imported numbers; schema overrides and sticky usedness are either
represented exactly or refused before mutation; ambiguous commit is
reconciled; plaintext is zeroed; trusted metadata is not confused with adapter
defaults; and a source hash proves the original file was untouched.
Every migrated account reopens as `watch-chain`, and duplicate legacy scripts
are either merged losslessly once or refused with their dedicated reason. Raw
transaction record-key mismatches fail immutable preflight for confirmed and
unmined fixtures. Corrupt credit amounts and indices, debit amounts and input
indices, debit outpoints, and one-sided spent-credit pointers fail the same
immutable preflight. Missing, extra, malformed, and misdirected mined-balance,
block-membership, unspent, and unmined-input index fixtures fail before the
destination is opened for mutation.
Wrong-network fixtures with contradictory persisted evidence return the typed
refusal before destination mutation. The trusted custom-signet challenge is
written into [Task 340][t340] identity metadata, but an empty legacy source with equal
truncated magic is not treated as proof of a different historical challenge.
Zero-counter BIP49Plus/BIP84 fixtures with equal normalized XPub payloads fail
before branch derivation or destination mutation. Plaintext decoder exposure
remains bounded on success, refusal, and cancellation.

## Definition of done

- A supported legacy wallet converts into one fresh SQLite destination with
  equivalent identity, custody, accounts, scripts, transaction graph, leases,
  and sync state.
- Every unsupported or corrupt source shape returns a typed refusal before
  destination mutation, and ambiguous import results reconcile exactly.
- The source remains an untouched rollback artifact and no PostgreSQL
  converter, generic import manifest, or new kvdb format is added.

[t329]: 329-atomic-wallet-account-creation.md
[t332]: 332-watch-only-script-policy.md
[t337]: 337-imported-xpub-derivation.md
[t340]: 340-durable-database-identity.md
[t345]: 345-durable-watch-registration.md
[t353]: 353-retire-modern-kvdb-backend.md
