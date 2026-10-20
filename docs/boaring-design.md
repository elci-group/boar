# BOARING — superseded by the real implementation

Status: **superseded**. This doc was drafted before discovering that the
real BOAR implementation — and a whole related "Baby/Boar/Boarish" ecosystem
— already exists at `/home/sal/baby` (repo `elci-group/baby`), separate from
this checkout. `/home/sal/boar` only ever held README/metadata for that
project; it was never the source tree.

That ecosystem already had a local, generic, content-addressed compilation
cache under the name **`boaring`**, and a Cargo-aware layer on top of it
named **`boarish`** — both doing most of what this doc originally proposed
under "Milestone 1," just under different names and already shipped. The one
genuine gap was git-hosted *remote* distribution, which didn't exist
anywhere in that workspace.

That gap is now implemented:

- `/home/sal/baby/crates/boar-index/src/remote.rs` — `GitRemote`: git-backed
  content-addressed store. A remote's local clone directory doubles as a
  `boaring::ArtifactStore`, so publishing reuses `boaring::Cache::insert`
  unchanged, and scanning reuses the same manifest format. Partial retrieval
  via `git clone --filter=blob:none --sparse` (manifests only, objects
  fetched on demand), falling back to a plain clone when a host doesn't
  support partial clone.
- `/home/sal/baby/crates/boarish/src/remote.rs` — `RemoteResolver`: wraps
  the existing local cache with a remote lookup on a local miss (fetch
  instead of recompile) and a best-effort push on a full miss.
- `/home/sal/baby/crates/boarish/src/config.rs` — global remote config
  (`~/.config/boarish/config.toml` + `BOARISH_REMOTE_*` env vars).

Differences from this doc's original proposal, and why:

- **No separate machine-key storage partition.** `boarish`'s existing
  `CompilationIdentity` hash already includes the rustc version and target
  triple, so two machines with different toolchains already get different
  cache keys by construction — an extra partition would have been redundant.
- **No `RUSTC_WRAPPER` integration.** `boarish` already intercepts a layer
  up, at the Cargo-invocation level (`boarish build` / `boarish cargo ...`
  wrapping `cargo` the way `boar` itself wraps `cargo`), so there was no
  compiler-invocation interception left to build — that milestone was
  already done under a different name.
- **Git-native CAS assumption held.** The core idea — git's own object
  store and partial-clone/sparse-checkout as the distribution mechanism,
  no host-specific API — carried through into the real implementation
  unchanged.

See `/home/sal/.claude/plans/polished-juggling-mitten.md` for the plan this
was built from.
