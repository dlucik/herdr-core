# hc fork runbook: own build with `send --if done`

Branch `local/v0.1.18-send-if-done` = tag `v0.1.18` + the fix (`feat: allow send --if done`)
+ version `0.1.18-sendifdone`. Not for upstream. The upstream PR is on `feat/send-if-done`
(from `master`).

## Why the version is `0.1.18-sendifdone`, not `0.1.18+sendifdone`

`hc update` compares versions with `older()` (`crates/herdr-core/src/cli/update.rs`): it splits
on `.` and `-` and takes three numbers. `0.1.18-sendifdone` reads as `0.1.18`, so:

- `hc update` (no args) says "up to date" until the next official release, then installs it;
- going back to official `0.1.18` works with `--force`.

With `+` the third part (`18+sendifdone`) does not parse, the build is never "older" than
anything, and `hc update` would stay on the fork forever without saying so.

Known test difference: `tests/update.rs`
`an_up_to_date_binary_restarts_only_an_older_running_core` fails on this branch, because it passes
`CARGO_PKG_VERSION` to `hc update --version`, which takes only `X.Y.Z`. It comes from the version
suffix, not from the fix; the rest of the suite passes (401 passed, 3 ignored, 2026-11-07).
The upstream branch `feat/send-if-done` has no suffix and passes `just check` in full.

## Build

    cd <worktree local/v0.1.18-send-if-done>
    git submodule update --init --depth 1        # vendor/ghostty
    cargo build --release --locked               # needs zig 0.16 on PATH
    target/release/herdr-core --version          # hc 0.1.18-sendifdone

## Swap in (not done by the agent; Łukasz decides when)

`hc update --from` stages, checks and swaps the binary, then restarts the core the planned way:
running terminals are kept.

    cp ~/.local/bin/herdr-core ~/.local/bin/herdr-core.0.1.18.bak   # optional fallback copy
    ~/.local/bin/herdr-core update --from <worktree>/target/release/herdr-core
    hc --version                                                    # hc 0.1.18-sendifdone
    hc server status                                                # version 0.1.18-sendifdone

Check: `hc send <t> "x" --if done` on a terminal in `done` types; `--if blocked` is refused
at argument parsing; `--if idle` on `done` gives `status_mismatch`.

## Back to official

    hc update --version 0.1.18 --force     # back to the official 0.1.18 now
    hc update                              # or: wait for the next release, it replaces the fork

`hc update` without args keeps the fork until a release newer than 0.1.18 exists. **That release
has the fix only if the maintainer merged the PR** (watch the upstream PR / issue #154). If a new
release comes without the fix, rebuild: rebase this branch on the new tag, bump the version to
`<new>-sendifdone`, build, `hc update --from …` again.

## Remote machines

`hc machine add` copies "this hc" to a machine of the same OS/CPU; for another CPU it downloads
the release of the same version, and `0.1.18-sendifdone` has no release: it fails with
`install_failed` ("has no release to download"). Install hc there yourself or use official.
