# WalletKit ImportAccount under ADR 0012

> **Status:** This document is decision support. It is not an executable
> roadmap task or an accepted ADR, and it does not override the task files,
> [QUEUE.md](../QUEUE.md), [ACTIVE.md](../ACTIVE.md), or accepted ADRs.

## Question

lnd's WalletKit `ImportAccount` imports an account extended public key into
lnd's active on-chain wallet. The account can derive addresses and track funds,
but it has no local private key. Today the kvdb wallet permits that account to
coexist with spendable seed-derived accounts. The SQL wallet, by contrast,
enforces ADR 0012: a spendable wallet rejects public-only account material.

That difference does not prevent basic lnd startup, ordinary channel operation,
or BOLT compliance. `ImportAccount` is an optional WalletKit feature. It does,
however, create a feature-parity gap for users who expect an imported account
to join the active wallet's address, balance, PSBT, and channel-funding flows.

The architectural choice is whether to:

1. amend ADR 0012 and permit account-level mixed custody in the existing
   btcwallet;
2. make lnd own and route to a second btcwallet in the same process;
3. put the watch-only wallet in a separate lnd or wallet deployment; or
4. retain the SQL rejection as a documented compatibility limit.

## Current behavior and terminology

The [lnd key import guide][key-import] describes an imported account as an
account XPub at `m/purpose'/coin_type'/account'`. WalletKit can derive its
external and internal branches, generate receiving addresses, track later
deposits and spends, and fund PSBTs. An external signer supplies the private-key
operation. The guide also says that events before import aren't found by the
current import flow because historical rescans aren't supported there.

The [`ImportAccount` RPC contract][import-rpc] carries a name, account XPub,
optional master fingerprint, address type, and dry-run flag. A dry run returns
sample external and internal addresses without installing the account. The RPC
describes an import into "the wallet," meaning the wallet behind the running
WalletKit service. It does not name a second wallet or return a wallet selector.

The [seeded-wallet integration scenario][import-itest] makes the compatibility
expectation concrete. A normally initialized node imports another seeded
node's account XPub. The importing node then:

- derives matching receiving addresses from that account;
- observes account-scoped confirmed and unconfirmed balances;
- selects the account's UTXOs and creates change with `FundPsbt`;
- sends the PSBT to the key-owning node for signing;
- publishes the signed transaction; and
- funds, operates, and closes a channel through the PSBT funding shim.

This is stronger than "lnd starts with SQL" or "WalletKit accepts the RPC." Full
compatibility means the imported account participates coherently in the active
wallet's account, transaction, PSBT, and channel workflows.

### Three account families must remain distinct

An ordinary imported account is an arbitrary, numberless XPub identity. The
current roadmap preserves that model in
[Task 337](../tasks/337-imported-xpub-derivation.md): public derivation selects it
by semantic account identity, while local private signing must fail.
[Task 392](../tasks/392-discovery-mode-account-creation.md) assigns this existing
compatibility path `watch-chain` discovery mode.

An exact path-bound external account is different. It carries a known hardened
account component and caller-declared origin information, but it is not derived
from the local root. [Task 387](../tasks/387-account-creation-identity-adr.md) and
[Task 391](../tasks/391-path-bound-watch-only-xpub-accounts.md) keep provenance and
path identity separate and currently admit this shape only in a uniformly
watch-only SQL wallet.

Remote-signer account families are different again. They use exact lnd key
family paths, such as purpose 1017, and need durable family identity and an
external signing contract. Ordinary `ImportAccount` must not be reinterpreted
as a remote-signer family or assigned a fabricated account number.

Finally, watch-only wallet initialization is not account import. Initializing a
watch-only shell establishes the custody model for the whole wallet before its
accounts are installed. Calling `ImportAccount` on an already seeded wallet
adds an account to an existing, spendable wallet. Replacing the latter with the
former changes the service and lifecycle contract.

## Evaluation criteria

Each approach is judged against two separate targets:

- **Basic lnd operation:** the node starts, its seeded wallet signs, and normal
  Lightning operation continues without imported-account support.
- **Complete WalletKit compatibility:** the seeded-wallet import scenario and
  its account-scoped address, balance, PSBT, and channel behavior remain
  available with the SQL backend.

The preferred ownership boundary should be the smallest one that preserves the
chosen target. If full compatibility is required, that means avoiding a new
wallet boundary unless lnd deliberately wants multi-wallet behavior as a
product feature.

## Approach 1: mixed accounts in the existing btcwallet

This approach amends ADR 0012 so a spendable SQL wallet may contain both
seed-derived accounts and explicitly public-only imported accounts. WalletKit
keeps using the active wallet, matching its present model.

### Required work and behavior

| Area | Required treatment |
| --- | --- |
| btcwallet | Amend ADR 0012, admit the precise mixed-account shape, persist account custody and provenance, derive public children while locked, and reject every local secret operation for public-only accounts. Keep ordinary numberless, exact path-bound, and remote-signer identities distinct. |
| lnd | Ideally no routing change. Validate whether current account selectors and error translation preserve account identity throughout WalletKit, Lightning `NewAddress`, PSBT, and funding-shim calls. Any gaps should be narrow account-aware fixes, not a second-wallet abstraction. |
| Data and storage | Keep one wallet, one transaction view, and one synchronization domain. Add or ratify immutable account-level custody facts and constraints. Existing wallet-wide `watch_only` state can no longer be the sole signing-authority test. |
| API behavior | `ImportAccount` succeeds against the active seeded wallet. `ListAccounts` reports the imported account as watch-only. Address generation, account balances, UTXO filtering, leases, and PSBT funding work. Local signing for its inputs or addresses returns a stable no-private-key or external-signing-required error. |
| Migration | Define how a kvdb seeded wallet that already contains imported XPub accounts maps into SQL without losing semantics. Existing SQL wallets need no synthetic second wallet, but schema changes may need backfill for seed-derived account custody. Reject ambiguous legacy rows rather than guessing. |
| Tests | Add custody truth-table tests, ordinary and path-bound identity tests, public derivation while locked, negative local-signing and message-signing tests, mixed-account balance and history tests, account-scoped coin selection and lease tests, PSBT fund/external-sign/finalize tests, recovery tests, and the seeded-wallet channel-funding scenario on SQL. |
| Compatibility | Preserves the current WalletKit model and offers the shortest path to complete parity, subject to a deliberate ADR change. |

### Security model

The wallet is spendable overall, but spendability is not inherited by every
account, address, or UTXO. Authorization must resolve the exact owning account
before secret lookup or signing. An imported account's XPub proves public
derivation capability, not private-key possession. Its child addresses may be
wallet-owned for tracking, coin selection, and PSBT metadata while remaining
locally unspendable.

That distinction must hold for direct signing, PSBT signing and finalization,
message signing by address, key derivation APIs, transaction authoring, and any
channel path that assumes wallet ownership implies local signing. No fallback
may map a numberless imported account to account zero or another seed-derived
account. Task 337 already states the key rule: a private signing request for an
imported identity must fail without falling through to another secret path.

### Implementation outline

1. Ratify an ADR amendment with an account-custody truth table and explicit
   exclusions for unsupported script-only or private-key shapes.
2. Make account custody and provenance immutable storage facts, then enforce
   them at public derivation, address ownership, and secret-resolution seams.
3. Audit every signer and finalizer path so authorization resolves the owning
   account before accessing secrets.
4. Enable ordinary numberless import in a seeded SQL wallet without changing
   exact path-bound or remote-signer semantics by accident.
5. Run focused btcwallet tests first, then the SQL-backed lnd WalletKit and PSBT
   channel scenario.

### Benefits and risks

The main benefit is architectural alignment: one active wallet continues to own
accounts, synchronization, transaction state, leases, and WalletKit behavior.
The change is concentrated in btcwallet's custody model rather than spread
across lnd.

The main risk is weakening a useful wallet-level invariant. Account-aware
authorization must be complete, not limited to `ImportAccount`. A missed
wallet-wide `watch_only` assumption could cause the wallet to request a secret
that cannot exist, report misleading capability, or select an input that the
chosen flow cannot finish. Those are reasons for a narrow ADR amendment and a
full signing-boundary audit, not reasons to label the existing rejection a bug.

## Approach 2: a second btcwallet inside the same lnd process

This approach keeps ADR 0012 unchanged. lnd's seeded wallet remains spendable,
while a second, uniformly watch-only btcwallet owns imported XPub accounts.

Creating that wallet is only the first step. WalletKit currently operates on
lnd's active wallet. It won't automatically send `ImportAccount`, `FundPsbt`,
balance queries, or notifications to another btcwallet. lnd would need a
first-class multi-wallet router and a clear aggregate-versus-selected-wallet
contract.

### Required work and behavior

| Area | Required treatment |
| --- | --- |
| btcwallet | Little or no change to ADR 0012. The Manager and SQL stores must support two concurrently loaded wallets with isolated identity, credentials, lifecycle, and storage. Shared chain ownership must match the accepted btcwallet architecture rather than inventing an lnd-only synchronization shortcut. |
| lnd | Add wallet identity and routing across account lookup, address derivation, balances, UTXOs, transaction history, PSBT funding, signing/finalization, leases, coin selection, notifications, recovery, startup, unlock, backup, and channel funding. Decide which wallet is "active" for every old RPC without a wallet selector. |
| Data and storage | Create a second wallet identity and account store. Define whether transaction and chain state are duplicated, shared, or aggregated; how outpoints and leases stay wallet-qualified; and how backups bind both wallets. |
| API behavior | Either extend APIs with wallet selectors, route by globally unique account identity, or present an aggregate facade. Existing requests that name only an account can become ambiguous. Signing must route seed-owned inputs to the primary wallet and reject or export watch-only inputs. |
| Migration | Decide whether existing kvdb imported accounts move to a companion wallet, how account names and history survive, whether historical discovery is possible, and how rollback works if one wallet migrates but the other doesn't. |
| Tests | Add two-wallet lifecycle, credential, lock-state, restart, backup, restore, chain replay, rollback, notification, aggregation, collision, routing, lease, mixed-input PSBT, and channel-funding tests. Re-run existing single-wallet suites to catch changed defaults. |
| Compatibility | Can reach parity only after lnd adopts multi-wallet routing as a supported architecture. Merely opening a second btcwallet does not provide WalletKit parity. |

### Routing and lifecycle blast radius

The following areas need explicit ownership rules:

- **Account selection:** account names can collide across wallets. Every
  account-bearing RPC needs a wallet-qualified selector or a global uniqueness
  rule. Default-account behavior must remain deterministic.
- **Address derivation:** `NewAddress`, `NextAddr`, and change allocation must
  reach the correct wallet. Dry-run derivation must not mutate either wallet.
- **Balances and UTXOs:** lnd must define whether legacy totals aggregate both
  wallets. Account filters, confirmation state, and spendability labels must
  preserve the owning wallet.
- **Transaction history:** one transaction may touch outputs from both wallets.
  Deduplication, labels, confirmations, conflicts, and invalidation need an
  aggregate view or wallet-qualified results.
- **PSBT funding:** coin selection, fee estimation, change, derivation metadata,
  and input locking must remain in one coherent ownership domain. Mixed-wallet
  input selection should be forbidden unless the protocol explicitly supports
  multi-wallet coordination.
- **Signing and finalization:** the primary wallet can sign its inputs, while
  the watch-only wallet cannot. The router must preserve partial signatures and
  never treat aggregate ownership as signing authority.
- **Leases and coin selection:** lease IDs and outpoints need one authoritative
  owner. Release, expiry, restart replay, and concurrent selectors must not
  diverge between wallets.
- **Chain synchronization:** both wallets need consistent block and mempool
  delivery, rollback ordering, rescan policy, and tip state. Duplicate filters
  and notifications must not produce double accounting.
- **Notifications:** transactions, confirmations, balances, and address events
  need wallet identity internally even if old RPC streams omit it. Ordering and
  deduplication become lnd responsibilities.
- **Recovery:** recovery windows, imported-account discovery, live watches, and
  birthday or frontier state must be coordinated. Current key imports only see
  events after import, so a companion wallet doesn't solve historical recovery
  by itself.
- **Startup, unlock, and backup:** lnd must create, load, unlock, lock, stop,
  back up, restore, and reconcile two wallets. A watch-only wallet may not need
  private unlock, but its durable state is still required for recovery.
- **Channel funding:** the funding manager and PSBT shim must carry wallet and
  account identity through verification and finalization. Close outputs and
  later sweeps need an explicit destination, since channel lifecycle funds may
  return to the primary wallet rather than the imported account.

### Implementation outline

1. Accept an lnd architecture decision defining wallet identity, default
   routing, aggregation, and backward-compatible RPC behavior.
2. Introduce a wallet registry and wallet-qualified internal types before
   starting a second btcwallet.
3. Route read-only account, address, balance, history, and notification paths.
4. Route stateful UTXO, lease, coin-selection, and PSBT paths with strict
   single-wallet transaction ownership.
5. Integrate startup, recovery, backup, and channel funding, then run a full
   two-wallet fault and restart matrix.

### Benefits and risks

ADR 0012 remains simple, and private material stays physically separated from
the imported XPub store. This approach is rational if lnd independently wants
multiple on-chain wallets, distinct operators, or broader wallet tenancy.

The cost is high. It changes lnd's wallet model, creates API ambiguity, and
makes cross-wallet aggregation and lifecycle correctness part of the daemon's
contract. Choosing it only to avoid one account-level custody exception would
move a narrow compatibility question into almost every wallet-facing lnd
subsystem.

## Approach 3: a separate watch-only lnd or wallet deployment

This approach also preserves ADR 0012. The seeded lnd remains unchanged. A
separate process initializes a uniformly watch-only wallet and installs the
external XPub there. The key owner or another service signs externally.

This is a different operational product from importing an account into an
already seeded node. The second deployment has its own API endpoint, data,
chain synchronization, authentication, backup, monitoring, and availability.

### Required work and behavior

| Area | Required treatment |
| --- | --- |
| btcwallet | Use the accepted watch-only shell and path-bound account flows. Ordinary numberless import support depends on the chosen public API and must remain distinct from exact external accounts. No mixed-custody amendment is needed. |
| lnd | Potentially no change if operators call each deployment directly. Any unified user experience belongs in an external coordinator or later lnd multi-node feature. The watch-only deployment's ability to run without channel keys and unrelated daemon duties must be validated rather than assumed. |
| Data and storage | Fully separate databases, wallet identity, chain state, credentials, logs, and backups. No in-process aggregate transaction or lease store exists. |
| API behavior | Clients choose the seeded node or watch-only service explicitly. The seeded node's WalletKit still rejects or lacks the imported account. Cross-service PSBT transfer is explicit. Channel funding on the seeded node requires a PSBT coordination flow rather than transparent active-wallet account selection. |
| Migration | Operators provision a new service and re-import XPub metadata. Pre-import history remains subject to rescan support. Existing account labels, generated-address cursors, leases, and transaction labels need an export/import contract if they must be preserved. |
| Tests | Add deployment bootstrap, authentication, watch synchronization, restart, backup/restore, external signing, PSBT transfer, failure recovery, and any cross-service channel-funding tests. Test outages and stale chain state between services. |
| Compatibility | Provides a useful watch-only workflow, but not transparent compatibility with `ImportAccount` on the seeded node's active WalletKit wallet. |

### Implementation outline

1. Define whether the second process is a full lnd node, a btcwallet service,
   or an operator-managed tool, and state which APIs it promises.
2. Provision a watch-only wallet using the dedicated initialization flow, then
   install ordinary or exact path-bound accounts through the matching API.
3. Define authentication, PSBT exchange, publication, lease cleanup, and
   failure ownership across the process boundary.
4. Add operational backup, restore, monitoring, and synchronized-chain tests.
5. If channel funding is required, specify and test the coordinator that joins
   the watch-only PSBT flow to the seeded lnd funding state machine.

### Benefits and risks

Process isolation gives the clearest custody and operational boundary. It can
fit organizations that already separate watch, signing, and Lightning roles or
that prefer a dedicated watch service over WalletKit parity.

It also shifts complexity to operators. There is no automatic combined balance,
transaction history, account list, lease domain, or channel-funding path. Calling
this complete `ImportAccount` compatibility would be misleading unless a
coordinator recreates those semantics and exposes them through the expected
endpoint.

## Approach 4: retain the SQL rejection

This approach keeps ADR 0012 as written and treats the difference from kvdb as
an intentional compatibility limitation. SQL-backed seeded wallets continue to
reject public-only imported accounts. Uniformly watch-only SQL wallets can use
the separate account shapes admitted by the current roadmap.

### Required work and behavior

| Area | Required treatment |
| --- | --- |
| btcwallet | Preserve early validation from [Task 332](../tasks/332-watch-only-script-policy.md). Keep the grandfathered kvdb mixed-mode import as an explicit legacy exception. Return a stable error before secret preparation or storage mutation. |
| lnd | Translate and document the unsupported SQL combination clearly. Capability discovery or backend-aware documentation may be useful, but must not pretend the RPC succeeded. No BOLT or startup behavior needs to change. |
| Data and storage | No new shape. SQL wallets remain uniformly spendable or watch-only. kvdb and SQL retain different import contracts until kvdb retirement or a later decision. |
| API behavior | `ImportAccount` on a seeded SQL wallet fails predictably. Basic lnd remains available. Complete WalletKit feature parity is not claimed. Watch-only initialization remains a separate supported flow where its own prerequisites are met. |
| Migration | A kvdb wallet containing mixed imported accounts cannot be converted silently under the SQL invariant. Migration must reject it, require an operator choice, or wait for a ratified split/export design. Dropping the account or fabricating private custody is not acceptable. |
| Tests | Add stable-error and no-mutation tests in btcwallet, lnd RPC error tests, backend capability documentation tests if available, and migration rejection fixtures for mixed kvdb wallets. Keep ordinary lnd startup and seeded-wallet operation coverage. |
| Compatibility | Preserves basic lnd operation but explicitly does not provide complete WalletKit `ImportAccount` compatibility on seeded SQL wallets. |

### Implementation outline

1. Ratify that wallet-level custody uniformity outranks seeded-wallet
   `ImportAccount` parity for SQL.
2. Freeze the rejection point and stable public error before mutation.
3. Document the backend difference in lnd-facing release and operator material.
4. Add migration preflight that identifies mixed kvdb wallets and offers only
   approved operator choices.
5. Test both the rejection and unaffected basic lnd behavior.

### Benefits and risks

This is the smallest implementation and keeps a strong, easy-to-audit custody
invariant. It is rational if SQL's product contract values wallet-level
uniformity more than this WalletKit feature or if no supported users depend on
seeded-wallet imports.

The tradeoff is real and user-visible. A valid WalletKit workflow supported by
kvdb fails on SQL, including the upstream seeded-wallet PSBT and channel-funding
scenario. The limitation must be explicit in compatibility claims and migration
tools.

## Decision matrix

| Approach | ADR 0012 | lnd change | Operational cost | Basic lnd | Full WalletKit parity | Main risk |
| --- | --- | --- | --- | --- | --- | --- |
| 1. Mixed accounts in active wallet | Amend narrowly | Low to medium, mostly validation | Low | Yes | Yes | Incomplete account-level signing authorization |
| 2. Second wallet in one lnd | Preserve | Very high, first-class multi-wallet routing | Medium | Yes | Possible after broad work | Ambiguous routing and split lifecycle state |
| 3. Separate watch-only deployment | Preserve | Low in core, high in integration | High | Yes | No, unless a coordinator recreates it | Operational split and misleading parity claims |
| 4. Explicit SQL limitation | Preserve | Low | Low | Yes | No | Permanent backend contract gap |

## Recommendation

If complete WalletKit compatibility is a product goal, choose approach 1: an
explicit, narrow ADR 0012 amendment that permits public-only imported accounts
inside the existing seeded wallet, backed by account-specific custody and
signing enforcement. This keeps ownership where WalletKit already expects it,
inside lnd's active wallet, and avoids inventing routing across synchronization,
transactions, leases, PSBTs, recovery, and channel funding.

The amendment should not make all public material legal in spendable wallets.
It should name the accepted account shapes, preserve immutable provenance, and
require exact account or address ownership before any signing decision. Ordinary
numberless imports, exact path-bound external accounts, and remote-signer
families must keep separate identities and admission rules.

Approach 2 is sound only if lnd wants multi-wallet routing for reasons beyond
this RPC and is willing to make wallet identity part of its internal and public
contract. Approach 3 suits an operator who values process isolation and accepts
an explicit cross-service workflow. Approach 4 is the right choice if the ADR
0012 invariant is judged more valuable than parity, provided release,
migration, and API documentation state the limitation plainly.

## Decisions that must be ratified

Implementation should not begin until the owning ADR or product decision
answers these questions:

1. Is complete WalletKit `ImportAccount` parity on a seeded SQL wallet a
   supported product requirement, or is basic lnd operation sufficient?
2. May a spendable SQL wallet contain an account that is wallet-owned for
   tracking but lacks local signing authority? If so, which account shapes are
   admitted and which remain forbidden?
3. Which immutable facts express account custody, provenance, optional account
   number, optional master fingerprint, scope, schema, and discovery mode?
4. What stable error distinguishes "wallet recognizes this input" from "wallet
   can sign this input," including message signing and PSBT finalization?
5. Can automatic coin selection choose public-only inputs only when the caller
   has selected that account and an external-signing PSBT flow, or are broader
   mixed-input PSBTs supported?
6. What is the migration disposition for an existing kvdb seeded wallet with
   imported XPub accounts: direct mixed-account conversion, explicit export to
   another wallet, or rejection pending operator action?
7. Does imported-account recovery remain forward-only from import, or must SQL
   compatibility include historical rescan and durable discovery semantics?
8. If a second wallet is chosen, what are the wallet selector, aggregation,
   startup, unlock, backup, notification, lease, and channel-funding contracts?
9. Which upstream lnd scenarios become release gates, especially the seeded
   account import, external PSBT signing, and PSBT channel-funding test?

These decisions should be recorded in accepted ADR and task scope before code
changes. This document estimates the ownership boundary; it does not allocate
that work.

## Sources

- [lnd key import guide][key-import]
- [WalletKit `ImportAccount` RPC contract][import-rpc]
- [lnd seeded-wallet account import integration scenario][import-itest]
- [Task 332: watch-only and script-import policy](../tasks/332-watch-only-script-policy.md)
- [Task 337: imported-XPub public derivation](../tasks/337-imported-xpub-derivation.md)
- [Task 387: custom and path-bound account identity](../tasks/387-account-creation-identity-adr.md)
- [Task 392: discovery mode in account creation](../tasks/392-discovery-mode-account-creation.md)
- [Archived Task 1000](../archive/tasks/1000-complete-btcwallet-sql-role-readiness.md),
  historical analysis only

[key-import]: https://github.com/lightningnetwork/lnd/blob/master/docs/key_import.md
[import-rpc]: https://github.com/lightningnetwork/lnd/blob/d932f48678228e337a7cc75ee4e864245e6e2b99/lnrpc/walletrpc/walletkit.proto#L146-L171
[import-itest]: https://github.com/lightningnetwork/lnd/blob/d932f48678228e337a7cc75ee4e864245e6e2b99/itest/lnd_wallet_import_test.go#L497-L624
