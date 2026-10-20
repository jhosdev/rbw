# Maintaining this fork

This fork exists to run a reviewed, pinned build of rbw with security fixes and, later, per-request
confirmation. These rules keep it that way. They are stricter than upstream on purpose: this binary holds
the keys to a password vault.

## Dependencies: pinned, updated only for a reason

- **`Cargo.lock` is the source of truth.** Every build and install uses `--locked`. Never run a bare
  `cargo update`.
- **Update a dependency only when one of these is true:**
  1. `cargo audit` reports an advisory that is **reachable** in the normal build graph
     (`cargo tree -i <crate> -e normal` prints a path).
  2. Bitwarden changes its API, login flow or KDF and rbw breaks without the update.
  3. A build fix we need (toolchain or platform).
- **Smallest possible change:** `cargo update -p <crate>` for the affected crates only. Leave `Cargo.toml`
  bounds alone unless they block the fix, and say so in the commit.
- **Cooldown:** adopt a version that has been published for **at least 14 days**, unless the advisory is
  being actively exploited. Most malicious releases (like the April 2026 `@bitwarden/cli` npm compromise)
  are caught within that window.
- **Read the diff** of every updated crate before committing (`cargo vet` or the crate's own diff between
  the two versions), with extra care for `build.rs`, proc-macros, and anything touching TLS, crypto, the
  filesystem or the network.
- **One commit per update**, `Cargo.lock` only, listing each RUSTSEC id fixed and the old → new versions.
  See `8e43396` for the format.

Unreachable advisories (lockfile-only, build-time only) are recorded in the audit notes, not "fixed" by
updating: every update is new code to trust.

## Weekly check (local, no CI with write access)

```sh
git fetch upstream && git log --oneline main..upstream/main   # anything new upstream?
cargo audit                                                     # known advisories
cargo tree -i <crate> -e normal                                 # reachability of each one
```

There is deliberately **no hosted CI with write tokens and no Renovate/Dependabot auto-PRs**: a CI pipeline
with write access is exactly how supply-chain attacks get in. Checks run locally.

## Building and installing

- Build only from a commit you have read: `cargo build --release --locked`.
- Install to a **root-owned** path, so user-level malware can't replace it:
  `sudo install -m 0755 target/release/rbw target/release/rbw-agent /usr/local/bin/`.
- Never install binaries built by someone else's CI, including this repo's GitHub releases. There are none
  on purpose.

## Upstream

Upstream is doy/rbw. We don't open PRs yet. When we do, each change goes as its own small PR. Upstream PR
#325 already covers part of the TLS update (`bytes`, `rustls-webpki` 0.103.10); our branch goes further
(`rustls` 0.23.46, `rustls-webpki` 0.103.16).
