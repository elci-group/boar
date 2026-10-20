---
name: boar
description: Run Rust Cargo commands through BOAR on Linux when adaptive tmpfs-backed build targets can reduce disk I/O while retaining automatic disk fallback. Use for Cargo build, check, clippy, doc, test, run, or bench work; do not use for non-Cargo projects, non-Linux hosts, nested BOAR invocations, or when build artifacts must persist across reboot.
---

# BOAR

BOAR is a Linux-only Cargo wrapper that selects a per-workspace target on
tmpfs when memory and pressure checks say it is safe, otherwise it runs Cargo
against the disk target. Prefer its default `auto` mode for agent work.

## Decide whether to use it

Use BOAR when all of these are true:

- the task runs Cargo in a Rust workspace on Linux;
- `boar` is installed and the process is not already running under BOAR;
- temporary build artifacts are acceptable; and
- the user has not selected a different target-directory strategy.

Use Cargo directly when `BOAR_ACTIVE=1`, when `boar` is unavailable, or when
the target must survive reboot. Do not install BOAR, create `.boar.toml`, or
change the user's Cargo configuration merely because this skill was selected.

When the environment is unfamiliar or BOAR fails a readiness check, run
`boar doctor`. Treat its warnings as decision input; do not force RAM mode to
bypass pressure, filesystem, or capacity safeguards.

When working on BOAR's own source, compare `boar --version` with the package
version in `Cargo.toml`. An installed binary may be newer or older than the
checkout; use the checkout's build and tests as evidence about source health,
not the installed binary's successful diagnostics.

## Run Cargo work

Use the direct form for common commands and pass Cargo flags after the
subcommand:

```bash
boar check --all-targets
boar test --workspace --locked
boar clippy --all-targets -- -D warnings
boar build --release
```

For another Cargo subcommand, use `boar cargo <subcommand> ...`.

Do not pass Cargo's `--target-dir`; BOAR rejects it because it conflicts with
placement. If disk placement is required for one invocation, use
`boar --mode disk <command> ...`. Reserve `--mode ram` and `--no-monitor` for
an explicit user choice or a measured environment where the risk is accepted.

BOAR may stop and retry `build`, `check`, `clippy`, and `doc` on disk after
runtime pressure. It deliberately does not retry `test`, `run`, or `bench`,
because repeating user code could duplicate side effects.

## Preserve boundaries

- Report the underlying Cargo exit status and relevant BOAR placement or
  fallback message.
- Do not assume `/dev/shm` is unconstrained by containers or cgroups; fall back
  to Cargo if BOAR cannot establish a safe budget.
- `boar clean` removes only the current workspace's marked RAM target.
- Treat `boar clean --all` as a broad destructive operation and require the
  user's explicit request before running it.
- Successful RAM targets are incremental caches, not durable deliverables.
