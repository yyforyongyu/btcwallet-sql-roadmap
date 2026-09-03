# Task 356 — defer change-script allocation until viable

Component: **TxCreator, PsbtManager, and Wallet-owned address storage**.

Type: **Bug fix**.

Dependencies: **[Task 344][t344], [Task 355][t355], [Task 357][t357],
[Task 404][t404], and [Task 467][t467]**.

## Rationale

Maintained `TxCreator.CreateTransaction` and `PsbtManager.FundPsbt` calls
prepare a change source backed by address derivation. The unchanged authoring
flow can request its script before establishing positive, non-dust change, so
a failed, no-change, or retried request can consume a Wallet address without
returning a transaction that uses it.

## Details

Prepare one non-durable change candidate containing the scope, account,
internal branch, expected index, address type, script, and script size needed
by the existing authoring call. Return that candidate's script through the
unchanged `txauthor.ChangeSource` interface. Discard it when authoring fails or
does not produce viable change.

For viable SQL change, commit that exact candidate through Task 344's private
watched-allocation transaction so the cursor, address, and durable watch
obligation remain atomic. For viable modern-kvdb change, use its existing
Wallet-owned count-one transaction before existing watch registration. Perform
either mutation before publishing the transaction or creating lease effects.
If the stored cursor no longer matches the expected index, discard the authored
transaction and candidate and retry with the caller's context. The retry must
not publish or lease the conflicted result. Only that exact precommit cursor
mismatch retries. After an ambiguous commit response, classify the exact
backend facts, including the SQL obligation: continue for the exact committed
candidate, return the original error for clean absence, and return
[Task 467](../archive/tasks/467-indeterminate-commit-identity.md)'s
`ErrIndeterminateCommit` for partial, conflicting, unreadable, or unavailable
state. Never retry an ambiguous response.

Keep the supported descriptor set closed to compressed P2PKH, P2WPKH, nested
P2WPKH, and key-path P2TR. Reject uncompressed P2PKH, unknown or
default-classed scripts, every other P2SH or P2WSH form, and Taproot
script-path shapes before authoring.

Exclusions: changes to `wallet/txauthor`, `waddrmgr`, or `walletdb`; copied fee,
size, dust, or viability arithmetic owned by dependencies; watched-change
delivery; PSBT metadata; leasing policy; and broader descriptor support.

## Suggested fix

Use the existing backend mechanisms without forcing structural symmetry:

- For SQLite and PostgreSQL, pass the prepared candidate through Task 344's
  private allocation transaction. Reuse its expected-cursor CAS to advance the
  cursor, insert the exact address, and append the durable watch obligation in
  one commit. Do not add a second SQL cursor/address mutation.
- For kvdb, preview from unchanged `AccountProperties`, then use an existing
  `walletdb` transaction and unchanged `NextInternalAddresses` behavior to
  compare and commit the exact candidate. An exact cursor mismatch rolls back
  the transaction, and unchanged `EvictDerivedAddresses` behavior removes the
  corresponding live-cache entry before retry. After an ambiguous response,
  use that same eviction to remove candidate cache traces and invalidate the
  account cache before classification, so any later attempt reloads durable
  state. A cleanly absent candidate may be derived again.

Keep candidate preparation and commit private to Wallet-owned adapters. Tasks
348 and 408 own wrapper-specific live watch delivery and publication. Do not
add a shared coordinator, durable reservation, or compatibility layer.

## Verification

Focused `package wallet` tests exercise public `TxCreator.CreateTransaction`
and `PsbtManager.FundPsbt` calls for authoring failure, dust change, no change,
viable change, an exact-candidate conflict, and ambiguous commit results. They
prove failed and no-change requests do not advance an address cursor, viable
change commits exactly its returned script, and SQL never commits that script
without Task 344's durable obligation. Only an exact cursor mismatch retries
before publication or lease effects, and partial or conflicting state returns
`ErrIndeterminateCommit`. The ambiguous cases also cover exact commit, clean
absence, unreadable, and unavailable state. For kvdb, prove the next recovered
call reloads the durable cursor: an exact commit advances it, while clean
absence may reuse the same candidate. Do not require a next allocation to
succeed after an indeterminate result. Cover the closed accepted and rejected
descriptor families through the same public paths, using each backend's
specified storage mechanism.

## Definition of done

Maintained transaction creation and PSBT funding consume no change address
unless viable change uses that exact address. SQL consumes Task 344's atomic
address-and-obligation mutation, while kvdb keeps its existing count-one
transaction. Conflicts retry without observable transaction or lease effects,
and no excluded package or dependency-owned calculation is changed or copied.

[t344]: 344-bulk-address-allocation.md
[t355]: ../archive/tasks/355-transaction-size-accounting.md
[t357]: 357-transaction-amount-validation.md
[t404]: 404-transaction-input-amounts.md
[t467]: ../archive/tasks/467-indeterminate-commit-identity.md
