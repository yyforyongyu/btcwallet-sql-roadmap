# Task 459 — assemble durable Wallets for Manager startup

Component: **Manager startup Wallet discovery and assembly**.

Type: **Gap fix**.

Dependencies: **[Task 334][t334], [Task 453][t453], [Task 456][t456], and
[Task 458][t458]**.

## Rationale

Removing public `Manager.Load` means startup must discover every durable Wallet
without caller-supplied per-Wallet runtime configuration. Manager needs one
complete deterministic set of Wallet candidates before Task 331 can start and
publish the aggregate runtime.

## Details

- Add one internal backend-neutral operation that enumerates every durable
  Wallet identity and assembles complete Wallet candidates from Task 456's
  immutable Manager runtime configuration.
- Return candidates in stable durable Wallet-ID order. SQL returns every Wallet
  row in the Manager database; modern kvdb returns its sole Wallet when
  initialized and an empty set otherwise.
- Construct candidates with wallet-scoped Store adapters, immutable local
  policy, Task 458's Wallet-owned call admission state, and Wallet-owned worker
  state. Inject no Manager context, Manager admission capability, per-Wallet
  SQL chain source, or Store-close authority.
- Keep every candidate unstarted. Do not call private Wallet start, register
  chain work, or publish an active-set entry; Task 331 owns startup,
  all-or-nothing Running publication, and exact active-pointer installation.
- Preserve Task 453's exact-pointer identity contract: the candidate handed to
  Task 331 is the same pointer that final successful publication exposes. Add
  no second cache owner, transient sentinel, per-name lifecycle entry, or
  shared or joinable assembly completion.
- Return one private, one-shot assembly outcome. Success contains the complete
  ordered candidate slice, no primary error, and no cleanup outcomes. Failure
  or cancellation contains no candidates, the exact primary error, and the
  attempt's cleanup outcomes separately; it is not a cache or operation
  registry.
- Own attempt-local cleanup before handoff. Track every component created by
  the assembly attempt and unwind only those resources in reverse ownership
  order on cancellation or failure. Normalize the reported cleanup outcomes by
  stable Wallet ID and fixed resource order independent of completion order, so
  Task 331 can record their errors without cleaning the resources again.
  Candidate cleanup cannot close the Manager-owned Store.
- Carry `context.Context` through enumeration, durable reads, candidate
  assembly, and final handoff. Task 331 admits one startup owner, so this task
  does not coordinate concurrent aggregate attempts or retain that context as
  Wallet lifetime authority.
- Treat an empty database as successful without reintroducing
  `ErrWalletNotFound`. Preserve stable durable corruption and enumeration-error
  classification; one malformed Wallet fails complete assembly rather than
  silently omitting it.
- Exclude private Wallet start, worker execution, chain registration, Manager
  state, Create admission, active-set publication, Store closure, public Load
  or lookup APIs, runtime manifests, and cross-process fencing.

## Suggested fix

Extend the Manager backend seam with stable durable-Wallet enumeration. Build
complete unstarted candidates from Task 456 policy plus wallet-scoped durable
identity, record attempt-owned components for reverse cleanup, and return one
private outcome that separates candidates or the primary failure from ordered
cleanup outcomes. Hand only the successful all-or-nothing slice to Task 331 for
start and exact publication.

## Verification

- Enumerate empty, one-Wallet, and multi-Wallet SQL databases on SQLite and
  PostgreSQL and prove stable durable-ID order and exact pointer identity.
- Cover empty and initialized modern kvdb databases without inventing a second
  Wallet or persistent alias.
- Inspect each candidate before handoff and prove its call tracker and worker
  state are initialized but no worker, private start, chain registration, or
  active publication has occurred.
- Place cancellation barriers before enumeration, between candidate
  assemblies, and immediately before handoff. Every losing path closes only
  attempt-owned components, returns no candidates, preserves its exact primary
  error separately from stable-ordered cleanup outcomes, and never closes
  Store.
- Inject one malformed durable Wallet among valid rows and prove assembly
  returns no partial set and leaks no component, secret, or goroutine. Fail
  multiple cleanup steps and prove every outcome is returned once in stable
  Wallet-ID and resource order.
- Hand the ordered candidates to Task 331 and prove successful publication uses
  those exact pointers only after every private start succeeds.

## Definition of done

- Manager can discover and assemble every durable Wallet from one immutable
  Manager configuration without public Load.
- The candidate set is complete, stable-ordered, unstarted, and all-or-nothing
  across kvdb, SQLite, and PostgreSQL.
- Each candidate owns its call tracker and worker state but no Manager context,
  lifecycle capability, SQL source, or Store-close authority.
- Cancellation or assembly failure returns its exact primary error separately,
  unwinds only attempt-owned resources, and reports every cleanup outcome in
  deterministic order. The PR adds no Manager lifecycle, active publication,
  public lookup, concurrent-attempt protocol, or cross-process behavior.

[t334]: ../archive/tasks/334-shared-chain-state.md
[t453]: ../archive/tasks/453-serialize-manager-wallet-publication.md
[t456]: ../archive/tasks/456-manager-runtime-configuration.md
[t458]: 458-manager-operation-admission.md
