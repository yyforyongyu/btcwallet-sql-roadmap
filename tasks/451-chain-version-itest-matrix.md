# Task 451 — run itests across supported chain releases

Component: **integration-test CI matrix and chain daemon provisioning**.

Dependencies: **None**.

## Rationale

The integration suite currently exercises one pinned btcd release and a
Bitcoin Core matrix that no longer matches the maintained upstream release
window. A passing test against one daemon release does not protect btcwallet
from RPC or notification differences across supported releases.

The CI harness needs a rolling compatibility matrix with an explicit support
policy. Every selected daemon release must run the same suite on kvdb, SQLite,
and PostgreSQL so a compatibility lane cannot hide a database-specific gap.

## Details

- Support the latest patch of Bitcoin Core's three maintained major release
  lines and the latest patch of btcd's current and immediately previous stable
  minor release lines.
- Start with Bitcoin Core `31.1`, `30.3`, and `29.4`, plus btcd `v0.26.2` and
  `v0.25.0`. Resolve future entries from the same policy when the matrix is
  updated.
- Run every selected Bitcoin Core and btcd release against kvdb, SQLite, and
  PostgreSQL. Do not reduce older-release lanes to SQLite or skip kvdb inside
  the suite.
- Run the complete maintained `make itest` suite in every selected lane. Keep
  btcd as the shared miner for bitcoind jobs, and use the latest btcd release
  for that role.
- Pin every selected daemon to its exact release using an immutable image
  reference or checksum-verified artifact, and verify the running daemon
  reports that release before starting the suite.
- Keep Neutrino coverage on the latest btcd miner only. Do not multiply its
  jobs across the btcd compatibility matrix.
- Retain all daemon provisioning work required to create and verify the
  selected lanes; unrelated provisioning work remains out of scope.
- Keep new wallet scenarios, new chain behavior beyond a required compatibility
  correction, unsupported daemon releases, and the historical pre-`v0.24.2`
  btcd fallback out of scope.

## Suggested fix

Express the btcd and bitcoind jobs as explicit CI matrix entries containing
the daemon release and database lane. Reuse the existing installers where
they can guarantee the exact release; otherwise add the smallest
checksum-verified provisioning step needed by the matrix. Include the daemon
release and database in job names and failure artifacts.

Document the rolling policy beside the matrix so an upstream release update
replaces an expired entry instead of growing the matrix indefinitely.

## Verification

- Prove the matrix expands to fifteen database lanes: three Bitcoin Core and
  two btcd releases, each paired with kvdb, SQLite, and PostgreSQL.
- Check each provisioned daemon's reported version before the integration
  suite runs, and fail on a mismatch.
- Run `make itest` for every selected daemon release on kvdb, SQLite, and
  PostgreSQL.
- Confirm Neutrino still runs only with the latest btcd miner and that Bitcoin
  Core `28.x` is no longer selected.
- Validate the workflow syntax and inspect the expanded job names and failure
  artifact names for an unambiguous daemon-and-database identity.

## Definition of done

- CI tests Bitcoin Core's three maintained major lines and btcd's latest two
  stable minor lines under the documented rolling policy.
- Every selected lane runs the full maintained integration suite using the
  exact expected daemon release.
- The matrix contains the complete selected-daemon-by-database cross product,
  and every lane runs the same maintained integration suite.
- Neutrino coverage is unchanged. Wallet behavior changes only to fix bugs
  revealed by selected lanes.
- The implementation PR contains only the CI compatibility outcome, required
  provisioning, and those fixes.
