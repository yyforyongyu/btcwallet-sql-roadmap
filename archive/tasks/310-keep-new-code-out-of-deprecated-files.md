# Task 310 — move this round's new code out of the deprecated files

Status: **HISTORICAL. Not binding.** Its standing rule has been superseded by
[[Task 314]] ruling 9 — *no voluntary deprecated-file changes; forced repair
only*, which **permits** the compile- and shared-code-forced edits this file's
byte-identical rule forbade. Read 314 ruling 9 for the active rule; everything
here is evidence.

> **Historical text; not operative.** The **zero-touch rule** at the end of this
> file was an owner directive and *previously* applied to all work. It is now
> superseded by [[Task 314]] ruling 9 — *no voluntary deprecated-file changes;
> forced repair only*. Everything else — the relocation proposal, the
> "move it to `wallet/passphrase_rotator.go` and keep one call-site edit"
> analysis — is **archived**: the later zero-touch directive forbids even that
> single edit, and [[Task 313]] never rebuilds the machinery, so nothing needs
> relocating. Read the rule first; treat the analysis above it as history.

Owner directive: **`wallet/deprecated.go` and its test files are no longer
maintained — leave them alone.** New declarations must not land there, and no
tracking surface (including `TODO` markers) should be opened in them.

This round violated that. The inventory and the fix are below. Verified against
`impl-secret-store` at `0d98a7d45` (`git diff 7bf664f83..impl-secret-store`).

## What this round added, and why

| File | Added this round |
| --- | --- |
| `wallet/deprecated.go` | **+34** |
| `wallet/deprecated_passphrase_guard_test.go` | **+46** |

The trigger was legitimate. T7 (review thread `UfRn0`) removed
`ChangePassphrases` from the exported `waddrmgr.AddrStore`, because declaring it
there breaks every external implementation at compile time. `walletLocker` in
`wallet/deprecated.go` held the **only** interface-mediated call site, so once
the method left the interface that expression stopped compiling. Something had
to change in that file.

What was not necessary was putting the *machinery* there. Landed inside
`deprecated.go`:

- `type atomicPassphraseRotator interface { ... }` — the extension interface.
- `var errAtomicRotationUnsupported = errors.New(...)` — the stable error.
- the type-assert guard in the `changePassphrases` case of `walletLocker`.
- a `TODO:` comment naming the interface, the assertion, the sentinel and the
  test for deletion with KVDB support.

That last item is also **self-inconsistent with this round's own decision.**
Task 305 dropped F2 with the explicit reasoning that a marker in a deprecated
method "would open a tracking surface for exactly the code this round declared
out of scope" — and then T7 added a `TODO` to the same file.

## The fix

Move the machinery to a new non-deprecated file, e.g.
`wallet/passphrase_rotator.go`, holding:

- `atomicPassphraseRotator`,
- `errAtomicRotationUnsupported`,
- a helper that performs the type assertion and returns the stable error, so
  the branch lives with the helper rather than in the deprecated loop.

`wallet/deprecated.go` then keeps a **single changed expression** and gains no
declarations, no guard block and no `TODO`:

```go
return rotatePassphrasesAtomic(w.addrStore, addrmgrNs, params)
```

Zero lines is not reachable while the deprecated path is the caller — the call
expression itself must change once the method is off the interface. One
changed expression is the floor, unless T7 is reverted (see below).

### The test moves too, and shrinks

`TestChangePassphrasesUnsupportedStore` currently drives `walletLocker` to
reach the guard, which is why it needs the `noAtomicRotationStore` fixture to
stub `IsLocked` (evaluated on every select iteration to build the `lockState`
send case) and `Lock` (called on the way out), plus a goroutine and
`wg.Add(1)`/`wg.Wait()` bookkeeping.

Testing the helper directly removes all of that: construct a store that does
not implement the interface, call the helper, assert the stable error. Keep the
falsification property that made the test worth having — with the assertion
removed the code panics on a nil interface, which is exactly what the stable
error replaces.

### Rejected alternative

Revert T7: re-declare `ChangePassphrases` on the exported `waddrmgr.AddrStore`.
This drops the deprecated delta to zero, but re-opens `UfRn0` — the exported
interface break for external implementers — which was reported fixed when that
thread was resolved. Not worth trading a real API break for one call line.

## Pre-existing deprecated churn — decide separately

Both files were already in the PR **before** this round, from the commit now
titled `waddrmgr+wallet: rotate passphrases atomically`:

- `wallet/deprecated.go` +71
- `wallet/deprecated_passphrase_guard_test.go` +106 (the file was created there)

That is the atomic-rotation wiring plus the C6 guard, and it is the reason F6
widened that commit's subject from `waddrmgr:` to `waddrmgr+wallet:`. It is not
this round's addition, so it is recorded here rather than assumed: **decide
whether that should also be pulled out of the deprecated surface**, or whether
the deprecated path keeps its atomic-rotation call because the alternative is
leaving a known non-atomic rotation in place.

Note the dependency: if that wiring is removed instead of relocated, the
`atomicPassphraseRotator` helper loses its only caller and both it and its test
can be deleted outright rather than moved.

## Cost

All three PRs (#1272, #1291, #1292) are green and pushed at `0d98a7d45` /
`8b4f29f74` / `1243c274c`. Doing this means force-pushing #1272 and rebasing
#1291 and #1292 on top, so CI re-runs on all three. It folds naturally into the
next round that touches #1272 for any other reason.

## Historical standing rule — superseded by [[Task 314]] ruling 9

**Do not touch deprecated files at all — not even to fix bugs.** They are
deprecated and we do not want to maintain them.

This supersedes the earlier, weaker version of this rule, which allowed "the
smallest possible call-site edit" when a supported-API change forced one. It
does not. Applies to `wallet/deprecated.go`, `wallet/deprecated_*_test.go`, and
`wallet/interface.go`'s aggregate `Interface`.

Consequences, stated so they are not rediscovered as surprises:

- **Known bugs there stay.** The two F2 defects on
  `Wallet.ChangePassphrases` — blocking forever on a never-started wallet, and
  reporting failure after the locker already accepted — are **permanent
  WONTFIX**, not deferred. Threads `Ue9wZ` and `UfRnj` carry the record.
- **Its non-atomic rotation stays.** The deprecated path rotating the two
  passphrases in separate calls is the very thing this round's atomic machinery
  was built to fix; that machinery is deleted, and the deprecated behaviour is
  left as-is.
- **If a supported-API change would break a deprecated caller's compile, the
  supported-API change is the thing that must adapt** — not the deprecated file.

### The rule is satisfied by construction, and it is checkable

[[Task 313]] never rebuilds the dual-rotation machinery, so nothing needs to
touch these files. At `sql-wallet` the only `ChangePassphrases` reference in
`deprecated.go` is the deprecated method's own declaration; every one of the
**240 added lines** this stack put into `wallet/deprecated.go` (+103) and
`wallet/deprecated_passphrase_guard_test.go` (+152) came from the work now being
deleted.

So the rebuild should leave both files **byte-identical to base**, which is a
one-command Definition-of-Done check:

```sh
git diff sql-wallet..<new-head> -- 'wallet/deprecated*'   # must be empty
```

## References

- [[Task 309]] — minimal public field surface; same review pass, different
  target. `ScriptIsSecret` and the two-passphrase API are tracked there.
- Task 305 — dropped F2 on the grounds that deprecated code gets no tracking
  surface; the `TODO` noted above contradicts it.
- Review thread `UfRn0` — the finding that made T7 necessary.
