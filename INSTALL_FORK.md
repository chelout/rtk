# Installing & Updating This Fork (`chelout/rtk`)

This is a personal fork of [`rtk-ai/rtk`](https://github.com/rtk-ai/rtk) with extra
features that are **not** in upstream:

- **Smart `grep`/`rg` rewrite** — translates grep flags into `rtk grep` form
  (`grep -rn pattern src/` → `rtk grep pattern src/`), strips no-ops, skips
  incompatible modes (`-l`, `-c`, `-q`, …), maps `--include`/`--exclude` to `--glob`.
- **Full-command tracking** — `rtk gain --history` shows the real reconstructed
  command instead of a bare `rtk grep`.

> ⚠️ **The most important thing to understand:** `brew install rtk` and
> `brew upgrade rtk` install the **official upstream** build from `homebrew-core`,
> **without this fork's features**. This fork is distributed only as source — you
> install it with `cargo install`, not brew. See the [Homebrew case](#case-b-rtk-was-originally-installed-via-homebrew-)
> below for how the two coexist.

For the standard/upstream install guide see [`INSTALL.md`](INSTALL.md).

---

## TL;DR

```bash
cd /path/to/rtk                 # this repo
git fetch upstream              # optional: pull latest upstream
git merge upstream/master       # optional: resolve conflicts if any
cargo install --path . --force  # build release + install to ~/.cargo/bin/rtk
rtk --version                   # verify
```

---

## Prerequisites

A working Rust toolchain. Verify:

```bash
cargo --version
```

If `cargo` is **not found** even though `~/.cargo/bin` is on your `PATH`, your
rustup shims are likely broken (`~/.cargo/bin/cargo → rustup → rustup-init`). Call
cargo from the installed toolchain directly:

```bash
~/.rustup/toolchains/stable-aarch64-apple-darwin/bin/cargo install --path . --force
```

(Replace `stable-aarch64-apple-darwin` with your toolchain — see
`ls ~/.rustup/toolchains/`.)

---

## Why not brew?

- This fork is **not published to any Homebrew tap**, so there is no bottle/formula
  for `chelout/rtk`.
- The `rtk` formula in `homebrew-core` points at **upstream `rtk-ai/rtk`**. Running
  `brew upgrade rtk` will replace your binary with the official build and silently
  drop the fork's features.
- `cargo install --path .` builds an **optimized release** binary and installs it to
  `~/.cargo/bin/rtk` — this is the canonical way to run the fork.

---

## Case A: rtk is NOT installed yet

```bash
cd /path/to/rtk
cargo install --path . --force
```

Make sure `~/.cargo/bin` is on your `PATH` (cargo usually adds this to your shell
profile on first install):

```bash
echo "$PATH" | tr ':' '\n' | grep -q "$HOME/.cargo/bin" && echo "ok" || \
  echo 'export PATH="$HOME/.cargo/bin:$PATH"  # add to ~/.zshrc or ~/.bashrc'
```

Verify (see [Verify you're running the fork](#verify-youre-running-the-fork)).

---

## Case B: rtk was originally installed via Homebrew ⭐

This is the case that needs care, because of **PATH shadowing**.

Homebrew links its binary at `/opt/homebrew/bin/rtk` (Apple Silicon) or
`/usr/local/bin/rtk` (Intel), and that directory is normally **earlier** in `PATH`
than `~/.cargo/bin`. So even after `cargo install --path .`, the **brew binary still
wins** unless you deal with the link.

### 1. Diagnose your current state

```bash
which rtk                  # which binary actually runs
ls -la "$(which rtk)"      # symlink? → Homebrew Cellar, or → ~/.cargo/bin/rtk?
brew list rtk 2>/dev/null  # is brew managing rtk?
rtk --version              # which version is live

# Which dir comes first in PATH (lower line number wins):
echo "$PATH" | tr ':' '\n' | grep -nE 'cargo/bin|homebrew/bin|/usr/local/bin'
```

Interpretation:

- `/opt/homebrew/bin/rtk → /opt/homebrew/Cellar/rtk/<ver>/bin/rtk` → a **clean brew
  install** (official upstream).
- `/opt/homebrew/bin/rtk → ~/.cargo/bin/rtk` → brew's link was **hijacked** to point
  at your cargo-built fork (this is option **B1** below).

### 2. Pick an approach

#### Option B1 — keep brew, point its link at the fork (lightest touch)

After building the fork, repoint Homebrew's symlink at the cargo binary:

```bash
cargo install --path . --force
ln -sf ~/.cargo/bin/rtk "$(brew --prefix)/bin/rtk"   # hijack brew's link
hash -r                                              # clear shell command cache
rtk --version
```

> ⚠️ **Caveat:** `brew upgrade rtk`, `brew reinstall rtk`, or `brew link --overwrite rtk`
> will **restore brew's official binary** and revert you to upstream. After any such
> brew operation, re-run the `ln -sf …` line above. To avoid accidental upgrades:
>
> ```bash
> brew pin rtk                       # brew upgrade will now skip rtk
> export HOMEBREW_NO_PATH_SHADOW_CHECK=1   # silence brew's "shadowed by" warning
> ```

#### Option B2 — unlink brew, rely on `~/.cargo/bin` (cleaner)

```bash
brew unlink rtk                 # removes the /opt/homebrew/bin/rtk symlink
cargo install --path . --force  # installs to ~/.cargo/bin/rtk
hash -r
rtk --version
```

Brew keeps the formula in its Cellar (so `brew upgrade` of other packages won't
error), but the **active** binary is your cargo build, found via `~/.cargo/bin`.
To go back to the official build: `brew link rtk`.

> If `~/.cargo/bin` is **not** earlier than the old brew bin dir in `PATH`, after
> `brew unlink` there is simply no brew binary anymore, so the cargo one is found.
> Confirm with `which rtk`.

#### Option B3 — uninstall the brew formula entirely (cleanest)

```bash
brew uninstall rtk
cargo install --path . --force
hash -r
rtk --version
```

No brew interference at all. To return to official upstream later: `brew install rtk`.

---

## Updating the fork later

```bash
cd /path/to/rtk
git fetch upstream
git merge upstream/master       # resolve conflicts if any
cargo install --path . --force
rtk --version && rtk gain
```

If you used **Option B1**, re-run the symlink line after upgrading if you also ran
any `brew` command in between:

```bash
ln -sf ~/.cargo/bin/rtk "$(brew --prefix)/bin/rtk"
```

---

## Verify you're running the fork

```bash
rtk --version                       # expect 0.42.0 or newer
which rtk && ls -la "$(which rtk)"   # path + symlink target
rtk gain                            # token-savings stats (sanity: not "command not found")

# Fork-only feature — smart grep rewrite (run outside this repo to avoid
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
# If installed via brew before:
brew link --overwrite rtk     # or: brew install rtk

# Or install upstream directly with cargo:
cargo install --git https://github.com/rtk-ai/rtk --force
```

---

## Quick reference

| Situation | Command |
| --- | --- |
| Fresh install of the fork | `cargo install --path . --force` |
| Update the fork after `git merge` | `cargo install --path . --force` |
| Repoint brew link → fork (B1) | `ln -sf ~/.cargo/bin/rtk "$(brew --prefix)/bin/rtk"` |
| Stop brew from reverting you | `brew pin rtk` |
| Unlink brew, use cargo (B2) | `brew unlink rtk && cargo install --path . --force` |
| Remove brew entirely (B3) | `brew uninstall rtk && cargo install --path . --force` |
| Back to official upstream | `brew link --overwrite rtk` or `brew install rtk` |
| `cargo` not on PATH | use `~/.rustup/toolchains/<triple>/bin/cargo` |

After any change, run `hash -r` (zsh/bash) so the shell forgets the old cached path,
then confirm with `which rtk` and `rtk --version`.
