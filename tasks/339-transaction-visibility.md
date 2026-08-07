# Task 339 — define public transaction visibility and status

Component: **TxReader**.

Dependencies: **None**.

## Rationale

Store transaction details carry pending, published, failed, replaced, and
orphaned status, but public readers currently omit it. Terminally invalid rows
can look like ordinary unconfirmed transactions.

## Details

- Compare the deprecated reader's visible behavior with library consumers.
- Preserve ADR 0006's user-visible audit history with a wallet-owned public
  status. Filtering retained terminal history requires an explicit ADR 0006
  amendment and another documented audit surface.
- Do not expose `wallet/internal/db.TxStatus`.
- Apply the SQL policy to both `GetTx` and `ListTxns`.
- Do not fabricate terminal history for kvdb. Its deprecated store exposes only
  the pending/published facts it actually retains; document that narrower
  backend result until kvdb is retired.

## Suggested fix

Add the smallest wallet-owned `TxStatus`, keep conversion centralized, and
table-test every persisted status. Do not expose internal DB enums or silently
hide SQL's retained failed, replaced, and orphaned attempts. Map only the
pending/published history that kvdb can prove.

## Verification

- Table-test `GetTx` and `ListTxns` conversion for pending, published, failed,
  replaced, and orphaned SQL rows.
- Verify terminal SQL history remains visible and no terminal row converts to
  ordinary pending state.
- Exercise kvdb pending and published fixtures and confirm the public contract
  neither fabricates nor promises terminal history for that backend.

## Definition of done

- `GetTx` and `ListTxns` expose wallet-owned status for every SQL state and
  only the pending or published facts that kvdb can prove.
- No internal database enum escapes the public API, and retained SQL audit
  history is neither hidden nor misclassified.
