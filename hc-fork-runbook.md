# hc fork runbook: own build with `send --if done`

Branch `local/v0.1.21-send-if-done` = tag `v0.1.21` + the fix (`feat: allow send --if done`)
+ version `0.1.21-sendifdone`. Not for upstream. The upstream PR is on `feat/send-if-done`
(from `master`).

## Why the version is `0.1.21-sendifdone`, not `0.1.21+sendifdone`

`hc update` compares versions with `older()` (`crates/herdr-core/src/cli/update.rs`): it splits
on `.` and `-` and takes three numbers. `0.1.21-sendifdone` reads as `0.1.21`, so:

- `hc update` (no args) says "up to date" until the next official release, then installs it;
- going back to official `0.1.21` works with `--force`.

With `+` the third part (`21+sendifdone`) does not parse, the build is never "older" than
anything, and `hc update` would stay on the fork forever without saying so.

## Build

    cd <worktree local/v0.1.21-send-if-done>
    git submodule update --init --depth 1        # vendor/ghostty
    cargo build --release --locked               # needs zig 0.16 on PATH
    target/release/herdr-core --version          # hc 0.1.21-sendifdone

## Swap in (not done by the agent; Łukasz decides when)

`hc update --from` stages, checks and swaps the binary, then restarts the core the planned way:
running terminals are kept.

    cp ~/.local/bin/herdr-core ~/.local/bin/herdr-core.0.1.21.bak   # optional fallback copy
    ~/.local/bin/herdr-core update --from <worktree>/target/release/herdr-core
    hc --version                                                    # hc 0.1.21-sendifdone
    hc server status                                                # version 0.1.21-sendifdone

Check: `hc send <t> "x" --if done` on a terminal in `done` types; `--if blocked` is refused
at argument parsing; `--if idle` on `done` gives `status_mismatch`.

## Back to official

    hc update --version 0.1.21 --force     # back to the official 0.1.21 now
    hc update                              # or: wait for the next release, it replaces the fork

`hc update` without args keeps the fork until a release newer than 0.1.21 exists. **That release
has the fix only if the maintainer merged the PR** (watch the upstream PR / issue #154). If a new
release comes without the fix, rebuild: rebase this branch on the new tag, bump the version to
`<new>-sendifdone`, build, `hc update --from …` again.

## Remote machines

`hc machine add` copies "this hc" to a machine of the same OS/CPU; for another CPU it downloads
the release of the same version, and `0.1.21-sendifdone` has no release: it fails with
`install_failed` ("has no release to download"). Install hc there yourself or use official.

## Release file for other computers (self-install, claude-voice-pilot #400)

A static Linux x86_64 build, packaged like upstream (`scripts/package`), with its SHA-256:

    rustup target add x86_64-unknown-linux-musl
    cargo build --release --locked --target x86_64-unknown-linux-musl -p herdr-core
    sh scripts/package target/x86_64-unknown-linux-musl/release/herdr-core dist/herdr-core-linux-x86_64.tar.gz
    (cd dist && sha256sum herdr-core-linux-x86_64.tar.gz > herdr-core-linux-x86_64.tar.gz.sha256)

The fork release tag has no leading `v` (`fork-0.1.21-sendifdone`): a `v*` tag would start the
upstream release workflow on the fork. Install on another computer:

    sha256sum -c herdr-core-linux-x86_64.tar.gz.sha256
    tar -xzf herdr-core-linux-x86_64.tar.gz
    ./herdr-core update --from ./herdr-core   # or put it on PATH as herdr-core, link hc to it
