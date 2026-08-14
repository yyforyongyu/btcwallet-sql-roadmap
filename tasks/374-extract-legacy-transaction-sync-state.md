# Task 374 — extract legacy transaction and sync state

Component: **Legacy migration transaction and sync extraction**.

Type: **Migration**.

Dependencies: **[Task 373][t373]**.

## Rationale

Legacy wtxmgr stores canonical transaction records alongside mined and unmined
indexes that can disagree after corruption. Its modern kvdb adapter deliberately
does not implement SQL's complete confirmed-history, replacement, or spend-edge
contract. A trustworthy importer needs one independently reconstructed graph,
not a sequence of ordinary adapter reads.

## Details

- Read only Task 335's private snapshot and consume Task 373's normalized
  address identities and sticky usedness. Do not access the original source or
  open a destination.
- Preserve the original pre-upgrade snapshot as the extraction authority. If
  an old source version requires maintained upgrade code for interpretation,
  upgrade only a separate scratch image and compare it with facts extracted
  from the untouched original; never replace or mutate the authoritative
  snapshot.
- Extract confirmed and unmined raw transactions, receive times, labels,
  coinbase and block membership, block order, pending/published state,
  credits, debits, UTXOs, spent edges, replacement/conflict facts, active
  leases with absolute expirations, birthday state, synced-to state, and the
  retained block history required to interpret them.
- Decode every transaction and require its computed txid to equal the
  independent legacy record key. Use checked monetary arithmetic for every
  output, aggregate, credit, and debit.
- Reconstruct the canonical graph from raw transactions and primary records.
  Require exact bidirectional agreement with credit/debit pointers, mined
  balance, block-membership indexes, unspent-outpoint indexes, and unmined-input
  spender indexes. Secondary indexes are evidence, not authority.
- Map legacy pending and published membership to SQL's closed status model.
  Preserve labels, receive times, change provenance, and supported conflict or
  replacement facts exactly.
- Require each credit and debit index, amount, outpoint, and spender pointer to
  resolve in both directions. Normalize a change flag only from its proven
  internal derivation and refuse a conflicting persisted flag.
- Prove sticky legacy address usedness equals the presence of a retained credit
  row in the normalized destination graph. Refuse true-without-credit and
  false-with-credit sources.
- Refuse multiple unmined spenders for one outpoint, duplicate txids, an active
  lease that cannot be represented exactly, unsupported status, or another
  historical shape that cannot fit the current SQL model. Do not select one
  spender or add BIP30 normalization.
- Bound allocations and return a typed resource-limit refusal instead of
  exhausting memory on an unsupported source. Keep every graph and sync type
  internal until Task 376 introduces the public result.
- Exclude destination mutation, rescanning instead of history import, generic
  manifests, BIP30 normalization, transaction-policy redesign, and caller
  cutover.

## Suggested fix

Add a migration-only wtxmgr reader that decodes primary records into an
internal normalized graph, derives all expected indexes from that graph, and
then compares each legacy index and aggregate exactly. Join credits to
Task 373's normalized script identities and return one complete internal
snapshot only after every graph, lease, usedness, block, and sync invariant
passes.

## Verification

- Representative mined, unmined, spent, replaced/conflicted, labeled,
  coinbase, leased, and reorg-retained histories round-trip into one normalized
  graph with the exact birthday and sync state.
- Use an old-version fixture with non-empty confirmed and unconfirmed history.
  Upgrade only a separate scratch image, extract from the preserved pre-upgrade
  image, and prove every original transaction survives later reopened SQL
  equivalence without source mutation.
- Confirmed and unmined record-key mismatches, output and aggregate overflows,
  invalid credit/debit indexes or amounts, wrong outpoints, and one-sided spent
  pointers return typed refusals.
- Missing, extra, malformed, and misdirected mined-balance,
  block-membership, unspent, and unmined-input index fixtures fail before any
  destination access.
- Multiple unmined spenders, duplicate txids, unsupported statuses,
  unrepresentable leases, conflicting change flags, and both directions of
  sticky-usedness disagreement are refused without source mutation.
- Cancellation and allocation-limit tests close iteration state and retain no
  partially published normalized snapshot.

## Definition of done

- A supported legacy history yields one internally consistent normalized
  transaction, UTXO, lease, block, and synchronization snapshot.
- Every independently stored identity, amount, edge, index, status, usedness,
  and sync fact is validated before destination access.
- Unsupported history is refused rather than discarded, rescanned,
  arbitrarily selected, or BIP30-normalized.
- The PR performs no source or destination mutation and exposes no public
  converter or generic import representation.

[t373]: 373-extract-legacy-account-address-state.md
