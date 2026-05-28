# Installing & Updating This Fork (`chelout/rtk`)

This is a personal fork of [`rtk-ai/rtk`](https://github.com/rtk-ai/rtk) with extra
features that are **not** in upstream:

- **Smart `grep`/`rg` rewrite** — translates grep flags into `rtk grep` form
  (`grep -rn pattern src/` → `rtk grep pattern src/`), strips no-ops, skips
  incompatible modes (`-l`, `-c`, `-q`, …), maps `--include`/`--exclude` to `--glob`.
- **Full-command tracking** — `rtk gain --history` shows the real reconstructed
  command instead of a bare `rtk grep`.

> ⚠️ **Two different `rtk` formulae exist — don't mix them up:**
>
> | Command | Installs | Has fork features? |
> | --- | --- | --- |
> | `brew install rtk` | `homebrew-core` → **upstream** `rtk-ai/rtk` | ❌ no |
> | `brew install chelout/rtk/rtk` | **this fork** (personal tap) | ✅ yes |
>
> They share the binary name `rtk` and **cannot be installed at the same time**.
> Running `brew upgrade rtk` on the upstream one will **not** give you the fork.

For the standard/upstream install guide see [`INSTALL.md`](INSTALL.md).

---

## TL;DR — install the fork via the personal tap

```bash
brew tap chelout/rtk
brew install chelout/rtk/rtk
rtk --version
```

Already have the upstream `rtk` from `homebrew-core`? See
[Switching from homebrew-core](#switching-from-an-existing-homebrew-core-rtk-) first.

---

## Install methods

There are two ways to run the fork. The Homebrew tap is recommended.

### Option 1 — Homebrew tap (recommended)

```bash
brew tap chelout/rtk
brew install chelout/rtk/rtk
```

- **macOS arm64** → installs a **prebuilt binary** (no Rust toolchain needed,
  installs in ~1s).
- **Other platforms** (macOS Intel, Linux) → **builds from source**; Homebrew pulls
  in `rust` as a build dependency automatically.

Update later:

```bash
brew update
brew upgrade chelout/rtk/rtk
```

### Option 2 — `cargo install` (build from source)

```bash
cd /path/to/rtk                 # this repo
git fetch upstream              # optional: pull latest upstream
git merge upstream/master       # optional: resolve conflicts if any
cargo install --path . --force  # build release → ~/.cargo/bin/rtk
rtk --version
```

Prerequisite: a Rust toolchain. If `cargo` is **not found** even though
`~/.cargo/bin` is on `PATH`, your rustup shims are broken
(`~/.cargo/bin/cargo → rustup → rustup-init`). Call cargo from the installed
toolchain directly:

```bash
~/.rustup/toolchains/stable-aarch64-apple-darwin/bin/cargo install --path . --force
```

(Replace the triple with yours — see `ls ~/.rustup/toolchains/`.)

> Note: `~/.cargo/bin` is usually **after** `/opt/homebrew/bin` in `PATH`. If you also
> have a Homebrew `rtk` linked, the brew one wins. With the tap (Option 1) this is a
> non-issue. With Option 2, either remove the brew link (`brew unlink rtk`) or ensure
> `~/.cargo/bin` comes first.

---

## Switching from an existing `homebrew-core` rtk ⭐

This is the Homebrew case that needs care. Because the upstream formula (`rtk`) and
the fork formula (`chelout/rtk/rtk`) **share the keg/binary name `rtk`**, they cannot
coexist — you must remove the upstream one, then install the fork.

### 1. Diagnose your current state

```bash
which rtk                  # which binary actually runs
ls -la "$(which rtk)"      # symlink? → Homebrew Cellar, or → ~/.cargo/bin/rtk?
brew list --versions rtk   # is brew managing rtk, and which version?
rtk --version              # which version is live

# Which dir comes first in PATH (lower line number wins):
echo "$PATH" | tr ':' '\n' | grep -nE 'cargo/bin|homebrew/bin|/usr/local/bin'
```

### 2. Switch to the fork tap

```bash
brew unpin rtk 2>/dev/null               # in case the upstream formula was pinned
rm -f "$(brew --prefix)/bin/rtk"         # ONLY if it's a manual symlink you created
                                         #   (e.g. → ~/.cargo/bin/rtk); brew won't
                                         #   overwrite a link it doesn't own
brew uninstall rtk                       # remove the homebrew-core keg
brew tap chelout/rtk
brew install chelout/rtk/rtk
hash -r                                  # clear the shell's cached command path
rtk --version
```

After this, `which rtk` → `$(brew --prefix)/bin/rtk` → `…/Cellar/rtk/<ver>/bin/rtk`
(the fork), and `brew upgrade chelout/rtk/rtk` keeps it current.

> **PATH shadowing:** Homebrew links its binary at `$(brew --prefix)/bin/rtk`, which
> is normally **earlier** in `PATH` than `~/.cargo/bin`. So a stray
> `~/.cargo/bin/rtk` from a past `cargo install` is harmlessly shadowed — but if you
> ever `brew uninstall` the tap formula, that older cargo binary becomes active
> again. Remove it (`rm ~/.cargo/bin/rtk`) if you don't want it.

### Alternative — skip Homebrew entirely (cargo only)

If you'd rather not use brew for rtk at all:

```bash
brew uninstall rtk            # drop the brew-managed one (any tap)
cargo install --path . --force
hash -r
rtk --version
```

---

## Verify you're running the fork

```bash
rtk --version                        # expect 0.42.0 or newer
which rtk && ls -la "$(which rtk)"    # path + (for brew) Cellar target
brew list --versions rtk             # e.g. "rtk 0.42.0-fork.1" when from the tap
rtk gain                             # token-savings stats (sanity: not "command not found")

# Fork-only feature — smart grep rewrite (run OUTSIDE this repo to avoid
# untrusted-filter warnings):
( cd /tmp && rtk rewrite 'grep -rn foo src/' )   # → rtk grep foo src/
```

> Note: `rtk rewrite` exits with code **3** ("ask" permission verdict) on a
> *successful* rewrite, **2** for "deny", **1** for "no rewrite", **0** for "allow".
> A non-zero exit on a successful rewrite is expected — it is the permission
> protocol the hook reads, not an error.

---

## Rollback to official upstream rtk

```bash
brew uninstall rtk        # removes the fork keg (named rtk)
brew untap chelout/rtk    # optional: drop the tap
brew install rtk          # reinstall homebrew-core upstream
```

Or install upstream directly with cargo:

```bash
cargo install --git https://github.com/rtk-ai/rtk --force
```

---

## Releasing a new fork version (maintainer)

The tap lives at [`chelout/homebrew-rtk`](https://github.com/chelout/homebrew-rtk)
(`Formula/rtk.rb`). To cut a new release:

```bash
# 1. Tag the fork (don't reuse upstream's vX.Y.Z; suffix it)
git tag -a vX.Y.Z-fork.N -m "…" && git push origin vX.Y.Z-fork.N

# 2. Build + package the arm64 prebuilt binary and attach to a GitHub release
cargo build --release
tar -czf rtk-aarch64-apple-darwin.tar.gz -C target/release rtk
gh release create vX.Y.Z-fork.N rtk-aarch64-apple-darwin.tar.gz --repo chelout/rtk

# 3. Compute the two checksums for the formula
shasum -a 256 rtk-aarch64-apple-darwin.tar.gz                 # prebuilt (arm64 url)
curl -fsSL https://github.com/chelout/rtk/archive/refs/tags/vX.Y.Z-fork.N.tar.gz \
  | shasum -a 256                                             # source (default url)

# 4. In chelout/homebrew-rtk, bump Formula/rtk.rb: version, both url tags,
#    both sha256 values. Validate, then commit + push:
brew style chelout/rtk/rtk && brew audit chelout/rtk/rtk
```

---

## Quick reference

| Situation | Command |
| --- | --- |
| Install the fork (recommended) | `brew tap chelout/rtk && brew install chelout/rtk/rtk` |
| Update the fork (tap) | `brew update && brew upgrade chelout/rtk/rtk` |
| Switching from upstream `rtk` | `brew unpin rtk; rm -f "$(brew --prefix)/bin/rtk"; brew uninstall rtk; brew install chelout/rtk/rtk` |
| Install the fork (cargo) | `cargo install --path . --force` |
| Update the fork (cargo) | `git merge upstream/master && cargo install --path . --force` |
| `cargo` not on PATH | use `~/.rustup/toolchains/<triple>/bin/cargo` |
| Back to official upstream | `brew uninstall rtk && brew install rtk` |

After any change, run `hash -r` so the shell forgets the old cached path, then
confirm with `which rtk` and `rtk --version`.
