# Task 311 — single encryption is the invariant for every backend

Status: **doctrine. Decided. Not independently executable.** Incorporated by
[[Task 313]] (delivered, now historical) and carried forward by [[Task 314]],
**the sole execution plan**. One statement below is amended by 314 §5 — the
waddrmgr-vault bullet, marked inline.

## The decision (owner)

**Every database backend assumes a single encryption key.** This holds for kvdb
as well as SQL. A **new** wallet created on kvdb uses single encryption from the
start — it does not get waddrmgr's historical public/private key pair.

**Public metadata is stored in the clear; secrets stay encrypted.** This
resolves the previously open Option A / Option B question in favour of
**Option A**.

**Existing kvdb wallets are not converted in this round.** `Manager.Load` must
return a stable migration-required error for a legacy-dual wallet. The
credential-gated converter is a future PR, recorded below so its constraints are
not rediscovered.

ADR 0012's grandfathering of kvdb's dual-key model narrows to *pre-existing,
unconverted* wallets.

### What "single encryption" does and does not mean

It means **one passphrase and one caller-visible protection contract**. It does
*not* mean literally one AES key: envelope encryption inside the vault is an
implementation detail, and SQL already keeps distinct private and script data
keys.

What it removes is the **caller-selectable** protection classes — `CKTPublic`
versus `CKTScript` at the wallet/signer layer — and the second passphrase that
derives them.

## What becomes plaintext — metadata only

**Correction (gpt review finding 3, verified).** An earlier revision of this
file implied that a watch-only wallet's *scripts* become plaintext. That was
wrong and did not match the SQL model it claimed to follow. Canonical SQL keeps
full script bodies in `encrypted_script`, and **every** `WalletVault` —
watch-only included — holds a script key:
`selectUnlockedCryptoKey` maps both `CKTScript` **and** `CKTPublic` onto
`cryptoKeyScript` (`keyvault/vault_cipher.go:79-92`, whose comment says "Every
watch-only vault holds it").

So the split is:

**Plaintext after conversion — public metadata**, everything `cryptoKeyPub`
protects today:

| Material | Written at | Read at |
| --- | --- | --- |
| Root / master extended **public** key | `manager.go:2568` | `manager.go:993` |
| Coin-type extended public keys (per scope) | `manager.go:2348` | `scoped_manager.go:761`, `:782` |
| Account extended public keys | `scoped_manager_accounts.go:58`, `:197`; `scoped_manager.go:2102`, `:2206`; `manager.go:2360` | `scoped_manager.go:550`, `:584`, `:807` |
| Imported address public keys | `scoped_manager.go:2430` | `scoped_manager.go:1100` |
| Address hashes / script **identities** | `scoped_manager.go:1488`, `:1738`, `:2713` | `scoped_manager.go:1133`, `:1148` |

**Still encrypted — secrets**: private keys, and **full redeem / witness /
taproot script bodies**, under the canonical script protection. A script body is
spend-relevant material, not metadata.

The SQL store already keeps the metadata plaintext (`MasterPubKey`
`data_types.go:253`, account `PublicKey` `:477`/`:514`, `ScriptPubKey` `:820`,
`PubKey` `:823`) while keeping script bodies encrypted, so this is convergence
onto the existing model rather than a new one.

**Accepted tradeoff.** Someone holding the `.db` file without a passphrase can
read xpubs and addresses. Fund safety unaffected; transaction-graph privacy is.
This is the exposure the SQL wallet already accepts.

### Why not Option B (metadata under the single key)

Recorded so it is not re-litigated. Sealing metadata under the one
passphrase-derived key would zero it on `lock()`, so a **locked** kvdb wallet
could no longer materialise account xpubs or imported/script addresses —
`decryptAccountPubKey` (`scoped_manager.go:807`),
`importedAddressRowToManaged` (`:1100`), `scriptAddressRowToManaged` (`:1133`,
`:1148`). Address enumeration and derivation while locked would break.

That gives kvdb a property SQL lacks (encrypted metadata) at the cost of one SQL
has (working locked reads) — the opposite of convergence. It is also not cheaper:
it rewrites every record too, just into different ciphertext.

## Format discrimination: reuse `mgrver`, add nothing

All three reviewers converge here, and it replaces the "explicit encryption
format marker" an earlier revision of this file and of 313 proposed:

- **Reuse waddrmgr's existing `mgrver`** as the format discriminator —
  `LatestMgrVersion` / `latestMgrVersion` (`db.go:28-33`), `putManagerVersion`
  (`db.go:2637`, `migrations.go:118`), and the `version < latestMgrVersion`
  check at `manager.go:2140`.
- **No** new format column, second registry, create manifest, completion marker
  or write lock.
- **Never infer a kvdb wallet's format from the configured backend alone.**
  Manager *admission* keys on the persisted version, not on `isKVDB`.
- **Two version domains — do not conflate them.** SQL wallets are written with
  `ManagerVersion: int32(waddrmgr.LatestMgrVersion)`
  (`wallet/runtime_store.go:296`), so SQL and kvdb carry the same number while
  meaning different things. A uniform "canonical means version N" test would
  either reject valid SQL wallets or admit legacy-dual kvdb ones. So: `mgrver`
  **inside the waddrmgr kvdb namespace** discriminates legacy-dual from
  canonical; SQL `wallets.manager_version` stays ordinary
  manager-compatibility metadata and is **not** repurposed as a format marker.
  The kvdb opener checks its namespace `mgrver` before migration/open; SQL opens
  its already-canonical representation.

### The `mgrver` trap — do not append an ordinary migration

Verified, and it is a data-corruption hazard rather than a style point (gpt
review finding 1):

- `LoadLegacyWallet` runs `migration.Upgrade(...)` at
  `kvdb/lifecycle.go:98` — **before** `waddrmgr.Open` at `:103`, so with no
  private credential available.
- The generic engine stamps the version itself once it has applied what it
  found: `mgr.SetVersion(ns, latestVersion)` at
  `walletdb@v1.6.0/migration/manager.go:152`, commented "we can now reflect the
  latest version upon the service".
- `waddrmgr.Create` writes `putManagerVersion(ns, latestMgrVersion)`
  (`db.go:2637`).

So **appending the single-encryption version to waddrmgr's migration list would
mark every legacy-dual wallet as converted without touching its ciphertext**,
and **merely bumping `latestMgrVersion` would make existing legacy callers create
the dual layout stamped with the new version**. Either one silently corrupts the
format contract.

Required shape instead:

- Keep the legacy migration list ending at its current latest version; reserve
  the **next** version for the credential-aware single format, reachable only
  through a distinct create/open path.
- Read `mgrver` *before* invoking `migration.Upgrade` or the legacy `Open`.
- Preserve legacy dual create/open for existing direct waddrmgr consumers
  (including the deprecated `Loader`).
- Never let the credential-free migration runner advance a wallet to the single
  version; only the future converter writes it, and only after the data rewrite
  succeeds.
- Do not change SQL's use of `waddrmgr.LatestMgrVersion` just to encode a kvdb
  encryption-format transition.

## Migration of existing wallets — OUT OF SCOPE, DEFERRED

**Explicitly out of scope for this round and every task in this set** (owner,
2026-07-30). It is named here only so the intended *solution direction* for
existing wallets is on record. It is **not** designed, specified, scheduled, or
part of any Definition of Done.

**What this round actually does:** new kvdb wallets are created in the
single-encryption format. A legacy-dual wallet is refused by the modern
`Manager` with a stable, matchable error — the single-passphrase public API must
not silently open a wallet whose public and private credentials differ. The
deprecated `Loader` path keeps opening old wallets unchanged, so existing
applications are unaffected and nothing needs converting for this work to land.

**The eventual solution, mentioned and nothing more:** a credential-gated,
in-place dual→single conversion, with the kvdb→SQL migration as the longer-term
path. Whoever picks it up will start from two facts already established by
reading the code — a wallet opens *locked*, so re-encrypting private or script
material needs a credential the load path lacks; and legacy script bodies sit
under either `CKTPublic` or `CKTScript`, so they must be re-encrypted rather
than copied. Timing, idempotency, transaction size and watch-only credential
handling are deliberately left unwritten.

## Consequences once this lands

- `db.AddressSecret.ScriptIsSecret` leaves the shared contract entirely.
- `scriptCryptoKeyType` (`signer.go:835`) and the `CKTScript`/`CKTPublic` branch
  in `encryptTaprootScript` (`address_manager.go:966`) collapse.
- `keyvault.ChangePassphraseParams` becomes `{Old, New}`, removing the public
  half restored by Task 305's F0 — which this decision reverses. Legacy dual
  rotation belongs to the future converter.
- **`LegacyWalletVault` is kept** — the waddrmgr-backed, **single-passphrase**
  implementation of `keyvault.Vault`, used by both the canonical kvdb path and
  the deprecated loader (owner rulings 1, 11, 13; [[Task 314]] §5). It is not part
  of the dual machinery that dies here: it takes one passphrase, exposes no
  `CryptoKeyType`, and keeps its crypto-class choice private. It exists because in
  kvdb waddrmgr **holds the keys** and must stay unlocked for its own script
  imports — **custody**, not encryption format. Retiring it means moving custody
  out of waddrmgr; implementing kvdb store-secret methods is explicitly out of
  scope (ruling 11).

  *Historical:* an earlier revision of this file asserted the adapter is "never
  built at all", reasoning from the correct premise that nothing at `sql-wallet`
  constructs it. The premise holds; the inference did not. Owner-approved 313 H3
  option (a) outranks 313 R5.
- `selectUnlockedCryptoKey` loses its key-type switch.
- The atomic dual-rotation machinery goes, which moots [[Task 312]] §A (the
  incomplete locked-state wipe) and most of [[Task 310]] — the code containing
  those problems is never rebuilt.

## Compatibility to work through

- **The public passphrase becomes vestigial** for converted/new wallets. `Open`
  derives `masterKeyPub` solely to unseal `cryptoKeyPub`; with only plaintext
  metadata under it, the argument is dead. Deprecate it explicitly rather than
  ignoring it silently, with the manager version as the discriminator.
- **lnd co-writes the `waddrmgr` bucket** through a hardcoded namespace key and
  opens with a public passphrase set to the user's password — any on-disk change
  is coordinated, not btcwallet-local.
- **btcwallet's own daemon** passes a configured default public passphrase
  (`walletsetup.go:184`, `config.go:271`); that config surface should say it is
  ignored for single-encryption wallets.
- **Who opens which format — the three-way distinction.** An earlier revision of
  this file said both formats must open in "mixed fleets" while also saying the
  modern `Manager.Load` refuses legacy-dual. Those contradicted each other (gpt
  finding 8, codex correction 2). The intended split:
  - **modern `Manager`**: opens the single format only; a legacy version is
    refused *before* construction with an exported, matchable
    `wallet.ErrMigrationRequired`;
  - **legacy / deprecated APIs** (the `Loader` path): keep opening the old format
    unchanged during the transition, so existing applications are unaffected;
  - **future converter**: the only thing that opens the old format through a
    restricted, credential-gated migration path.

  So there is no per-row key-type branch *and* no dual vault in the modern
  runtime — one admission check, then one vault.

## Settled scope question

kvdb wallet **creation stays** (owner, 2026-07-30) and new kvdb wallets are
single-encryption. The *implementation shape* is [[Task 313]]'s H3: option (a),
a waddrmgr single-encryption format whose canonical entry points write the
reserved next namespace `mgrver` (never a `latestMgrVersion` bump — see the
`mgrver` trap above), is **approved** (owner, 2026-07-30). The rejected
alternative was the canonical `WalletVault` over a kvdb wallet-secrets surface
that is `notImplemented` today.

## Suggested next step

**A small ADR 0014** (owner-approved), written as part of #1272 — deliberately
short and cross-referential, **not** a restatement of ADR 0009.

ADR 0009 is already "Single-Passphrase Encryption Model" and already rejects the
dual hierarchy, for the same reason we would give: downstream systems pin the
public passphrase to a constant, "which negated the privacy benefits while
keeping the complexity". What 0009 lacks is scope — its context is the SQL store
("works across SQLite and PostgreSQL"). So 0014 extends, it does not re-decide.

0014 states only what is new:

- 0009's model now binds **every** backend, including newly created kvdb wallets;
- the kvdb mechanism: canonical entry points write the **reserved next namespace
  `mgrver`** (9), the legacy list stays terminated at 8, and the credential-free
  migration path never advances a wallet to it;
- **kvdb holds exactly one wallet per database** — the second kvdb constraint,
  carried here alongside single encryption;
- the modern Manager **refuses** legacy-dual wallets with
  `wallet.ErrMigrationRequired`; conversion is deferred (one line, no algorithm);
- what is plaintext (public metadata) versus encrypted (private keys **and full
  script bodies**);
- Option B rejected, with the locked-read reasoning.

Repo precedent supports a new file over an in-place amendment: ADR 0013 narrows
0012 by saying so explicitly ("This ADR supersedes its SQL imported-address
bucket shape"). 0014 narrows ADR 0012's kvdb grandfathering the same way.

**Keep it at the invariant boundary.** Since migration is out of scope, the ADR
must **not** grow a converter algorithm, idempotency rules, transaction design or
a converter Definition of Done — an earlier revision of this section asked for
exactly that and contradicted the out-of-scope decision. Note the deferral and
the already-established credential boundary in one line each; a future migration
task owns the rest.

**It must also resolve two real contradictions with the existing ADRs** rather
than claiming alignment (gpt finding 7, both verified):

- **ADR 0009:37** says the simplified hierarchy works "by removing
  `CryptoPubKey` and `CryptoScriptKey`", while this task permits separate
  private and script data keys as envelope internals. State explicitly that one
  passphrase and one caller-visible contract do not forbid internally enveloped
  data keys, and that full script bodies stay encrypted.
- **ADR 0010** gives the keyvault ownership of lock state and encryption
  behaviour, while H3 option (a) delegates the concrete implementation to a
  modified waddrmgr behind a `Vault` adapter. State that this adapter is a
  temporary kvdb implementation of the `Vault` boundary — not permission for
  wallet callers to reach waddrmgr crypto classes — and that the kvdb→SQL
  migration removes it.

It should also record that `mgrver` dispatch and legacy refusal happen before
ordinary migration/open.

ADR 0014 **exists** at the current #1272 head
(`docs/developer/adr/0014-single-encryption-all-backends.md`). [[Task 314]] §7
and §9 own its final reconciliation edits — do not create another ADR.

## References

- [[Task 314]] — **the sole execution plan**; carries this doctrine forward.
- [[Task 313]] — delivered the single-encryption arc; **historical evidence**.
- ADR 0009 / ADR 0010 — the canonical single-passphrase vault model this aligns
  kvdb with.
- ADR 0012 — its kvdb grandfathering narrows to unconverted wallets.
