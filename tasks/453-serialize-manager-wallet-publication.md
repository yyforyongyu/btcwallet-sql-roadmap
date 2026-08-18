# Task 453 — serialize Manager Wallet publication

Component: **Manager Wallet cache publication**.

Type: **Bug fix**.

Dependencies: **None**.

## Rationale

Concurrent cold Loads for one wallet can both miss the Manager cache, assemble
separate Wallets, and race to publish different pointers. Create can likewise
commit a durable wallet and race Load before publishing its runtime pointer.
The overwritten assembly can leak attempt-owned resources, and callers can
receive different live Wallet identities for the same durable wallet.

This publication race is independent of starting or stopping a Wallet. It must
be closed before Task 368 attaches lifecycle coordination to the winning
runtime entry.

## Details

- Give each wallet name one Manager-owned runtime entry and publication gate
  shared by Create and Load.
- Serialize cold cache-miss Load assembly and publication. Concurrent successful
  Loads for the same name return the exact same Wallet pointer.
- Arbitrate Create and Load publication through the same entry after the durable
  wallet becomes visible. Publish one exact current pointer and never overwrite
  a winner.
- Close or release every attempt-owned assembly and resource that loses
  publication. Retained Store and legacy backend resources remain owned by
  Manager.
- Release a failed publication attempt so a later Create or Load can retry. Do
  not leave a failed entry or waiter permanently blocking the wallet name.
- Preserve backend parity. The legacy one-wallet kvdb path must reuse its
  Manager-owned opened data; cross-name kvdb admission remains Task 331's scope.
- Limit the entry to publication admission, the exact current pointer, and
  in-flight publication completion and result. Task 368 adds lifecycle state,
  coordination, terminal completion, and replacement after Stop.
- Exclude durable duplicate-create classification, Manager Close admission,
  Create and Load context cutoffs, Wallet Start and Stop behavior, generic
  operation registries, persistent manifests, and cross-process fencing.

## Suggested fix

Replace the bare Manager cache pointer with a small per-name runtime entry. Use
one entry-local publication guard or joined in-flight result so only one cold
assembly can become current while same-name callers join its result. Keep
blocking assembly outside the Manager-wide cache lock, and do not serialize
different SQL wallet names.

Have both Create and Load re-enter the same publication decision before
installing a pointer. Return the installed winner and deterministically release
every losing attempt. Do not add lifecycle transitions or a restart generation
to this task.

## Verification

- Barrier concurrent same-name cold Loads on kvdb, SQLite, and PostgreSQL. They
  return one exact pointer, perform one retained assembly, and leak no losing
  resource.
- Race Create after durable commit but before cache publication against Load.
  Both callers observe one exact current pointer, and neither can overwrite the
  winner.
- Force the winning cold assembly to fail, then prove waiting callers settle,
  all attempt-owned resources close, and a later retry can publish.
- Prove different SQL wallet names can assemble concurrently. Leave legacy
  kvdb cross-name admission to Task 331.
- Run the focused Manager publication races under
  `make unit-race pkg=wallet case=TestManagerWalletPublication`, followed by
  `make unit pkg=wallet`.

## Definition of done

- Cold Create and Load publication has one per-name owner and one exact winning
  Wallet pointer.
- Same-name callers join the winning publication, and every failed or losing
  attempt releases its resources and cannot strand the entry.
- The entry is independently usable before lifecycle coordination is added.
- The PR contains no lifecycle, Close, cancellation, durable duplicate-create,
  manifest, or cross-process behavior.
