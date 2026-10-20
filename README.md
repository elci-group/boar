# BOAR

**Build On Available RAM** is a RAM-aware Cargo wrapper for Linux. It places a
project's Cargo target directory on `tmpfs` when the machine has enough safe
headroom and falls back to disk when it does not.

Rust target directories are large, metadata-heavy, and disposable. BOAR keeps
that write load off the SSD without blindly treating all free memory as fair
game.

## What works in the MVP

- Reads `MemAvailable`, swap counters, and Linux PSI pressure signals while
  preserving an adaptive OS reserve (25% of RAM, with a size-aware 2 GiB
  floor).
- Accounts for both available system RAM and actual free space on the tmpfs.
- Persists per-workspace target estimates under the user cache directory and
  includes 25% growth headroom (at least 256 MiB). Existing targets are walked
  once to seed the cache, not on every invocation.
- Gives every Cargo workspace a stable, isolated target under
  `/dev/shm/boar/<project>-<hash>`.
- Waits for Cargo directly while an independent monitor watches retry-safe
  compilation commands. Memory PSI, active swapping, tmpfs exhaustion, or an
  exceeded target budget terminates the Cargo process group and retries on
  disk without adding polling latency to successful builds.
- Leaves successful RAM targets in place for incremental builds. The operating
  system clears them at shutdown/reboot because `/dev/shm` is tmpfs.
- Only deletes directories bearing BOAR's ownership marker.
- Uses the local `form3` (3form) crate for dependency-free terminal styling.

## Install

```bash
cargo install --path .
```

BOAR currently targets Linux and Rust 1.85 or newer.

## Use

```bash
boar doctor
boar build
boar test --workspace
boar run --release
boar cargo build --features gui
boar status
boar clean
```

Force a placement or cap BOAR's RAM budget:

```bash
boar --mode disk build
boar --mode ram build
boar --max-ram-mib 6144 build --release
boar --reserve-mib 4096 build
```

`boar clean` removes only the current workspace's RAM target. `boar clean
--all` explicitly removes all BOAR-owned RAM targets. It never removes the
disk fallback target.

BOAR passes arguments after the command directly to Cargo. A Cargo
`--target-dir` argument is rejected because it would bypass BOAR's placement;
use BOAR's `--ram-root` or `--disk-target` instead.

## Configuration

Run `boar init` inside a Cargo workspace to create `.boar.toml`:

```toml
mode = "auto"
# reserve_mib = 2048
# max_ram_mib = 8192
ram_root = "/dev/shm/boar"
disk_target = "target"
monitor = true
```

Precedence is CLI options, then environment, then `.boar.toml`, then defaults.
The equivalent environment variables are:

- `BOAR_MODE`
- `BOAR_RESERVE_MIB`
- `BOAR_MAX_RAM_MIB`
- `BOAR_RAM_ROOT`
- `BOAR_DISK_TARGET`
- `BOAR_MONITOR`
- `BOAR_CACHE_DIR`

If `disk_target` is unset, BOAR respects an existing `CARGO_TARGET_DIR` before
defaulting to the workspace's `target/`.

## Placement model

For automatic mode, BOAR calculates:

```text
memory budget = MemAvailable - adaptive reserve
tmpfs budget  = tmpfs free space - 128 MiB headroom
safe budget   = min(memory budget, tmpfs budget, optional user cap)
required      = max(existing target estimate + growth, 512 MiB)
```

It chooses tmpfs only when the safe budget covers the requirement and memory
PSI is below the preflight thresholds. Otherwise Cargo starts directly on
disk, so a constrained machine does not thrash or hit an out-of-space failure
first. High I/O PSI alone does not block tmpfs because that is precisely where
RAM-backed storage can help.

After a RAM build, BOAR updates the estimate from the change in tmpfs free
space. Disk targets are measured after the first or a substantial build. The
cache defaults to `$XDG_CACHE_HOME/boar/estimates` or
`$HOME/.cache/boar/estimates`; `BOAR_CACHE_DIR` overrides its root.

Live spillover is enabled for `build`, `check`, `clippy`, and `doc`, whose
compilation can be safely restarted. BOAR deliberately does not restart
`run`, `test`, or `bench`, because repeating user code or tests could duplicate
side effects. In forced RAM mode the same monitor stops and cleans an unsafe
build but does not spill, preserving the explicit placement choice.

## Scope and roadmap

The MVP implements a safe RAM-first/disk-spill target. File-level hybrid
tiering is not claimed: Cargo selects one target directory, so transparent
hot/cold file placement needs a filesystem layer (for example FUSE/overlay)
and substantially more failure handling.

Likely next steps are a long-running `board` daemon for concurrent allocation,
optional persistent dependency snapshots, filesystem-enforced per-project
quotas, and measured eviction policies. BOAR can also coexist with `sccache`:
BOAR accelerates active build files while `sccache` reuses compiler outputs.

## Safety notes

- Forced RAM mode still refuses to start below a 128 MiB safe budget.
- Automatic mode selects disk when memory PSI is elevated. Forced RAM mode
  also honors that guard while monitoring is enabled; `--no-monitor` is the
  explicit expert override.
- A custom RAM root must actually reside on `tmpfs` or `ramfs` in forced mode.
- Automatic cleanup is marker-guarded and constrained beneath the configured
  RAM root.
- `/dev/shm` consumes memory and may use swap under pressure; BOAR reduces risk
  but cannot override kernel memory policy or cgroup limits.

## License

MIT
