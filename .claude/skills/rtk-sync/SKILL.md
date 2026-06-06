---
description: Sync the chelout/rtk fork with upstream rtk-ai/rtk and cut a new -fork.N release. Checks the fork against the latest upstream release; only if upstream moved ahead does it merge, run the quality gate, then (after one confirmation) push master + tag, publish a GitHub release, and bump the chelout/homebrew-tap formula. Use for "rtk-sync", "sync the fork", "check upstream version", "release the fork", "/rtk-sync".
allowed-tools: Read Write Edit Bash Grep Glob
---

# rtk-sync — Fork ⇄ Upstream Sync & Release

Reconciles the `chelout/rtk` fork with the latest **upstream release** and — only
when upstream moved ahead — runs the full sync + release cycle end to end. This
skill is tied to this repo: it hardcodes `chelout/rtk`, `chelout/homebrew-tap`,
and the `vX.Y.Z-fork.N` tag scheme.

## When to use

- `/rtk-sync`, "sync the fork", "check if upstream released a new version",
  "release the fork".
- Periodic maintenance to keep the fork current with `rtk-ai/rtk`.

## Operating rules

- **Token efficiency is a priority.** The common case is "nothing new" — Phase 0
  must early-exit cheaply (~a handful of commands) and stop. Use targeted commands
  (`--oneline`, counts, `gh … --json <fields>`); pipe noisy output through
  `tail`/`rg`; never read large files whole; rely on the memories below instead of
  re-discovering facts.
- **One confirmation gate**, right before the first outward-facing action
  (Phase 5). Everything local — merge, conflict resolution, tests, commit, tag —
  runs without prompting.
- **Conflicts stop the run** for interactive resolution. Never auto-resolve blindly.
- Operate from the rtk repo root on `master` with a **clean working tree**. Refuse
  to start otherwise.
- `gh` defaults to upstream `rtk-ai/rtk` — **always** pass `-R chelout/rtk`.

## Environment (from memory — verify before relying)

- cargo/rustc are not on PATH here. Invoke via the toolchain bin:
  `TC="$HOME/.rustup/toolchains/stable-aarch64-apple-darwin/bin"; PATH="$TC:$PATH" "$TC/cargo" …`
  (memory `cargo-toolchain-path`). Also bypasses the rtk hook that rewrites bare `cargo`.
- The fork ships via the personal Homebrew tap `chelout/homebrew-tap`, formula
  `Formula/rtk.rb`, checked out at `$(brew --repository chelout/tap)` (memory
  `rtk-homebrew-tap`).
- The fork **CD workflow is red on every push** — benign (missing org secrets),
  not a blocker (memory `fork-cd-fails`).
- Canonical release commands live in `INSTALL_FORK.md` §"Releasing a new fork
  version (maintainer)" — this skill orchestrates them; that file is the source of
  truth if anything here drifts.

## Track progress

Once Phase 0 decides a sync is needed, create a TaskList for Phases 1–6. Skip the
task list entirely for the early-exit (up-to-date) case.

---

## Phase 0 — Version reconciliation (always runs; cheap; early-exit)

```bash
pwd                                   # must be the rtk repo root
git rev-parse --abbrev-ref HEAD       # expect: master
git status --porcelain                # MUST be empty — else STOP (dirty tree)
git fetch upstream --tags             # the `latest` tag "would clobber" warning is benign
```

Latest upstream **release** tag (pure `vX.Y.Z`; excludes `-fork`, `-rc`, `dev-`):

```bash
LATEST=$(git tag -l 'v*' --sort=-v:refname | rg '^v[0-9]+\.[0-9]+\.[0-9]+$' | head -1)
echo "latest upstream release: $LATEST"
```

Is anything new?

```bash
git merge-base --is-ancestor "$LATEST" HEAD && echo "UP-TO-DATE" || echo "BEHIND"
```

- **UP-TO-DATE** → report `Fork is current with $LATEST — nothing to sync.` and
  **STOP**. End here. This is the dominant path; keep it to the few commands above.
- **BEHIND** → continue. Set `VER=${LATEST#v}` (e.g. `0.42.3`).

> The merge target is the **release tag** `$LATEST`, not `upstream/master`, so the
> fork always tracks clean released versions (Cargo.toml carries the version across).

## Phase 1 — Pre-flight

```bash
ROLLBACK=$(git rev-parse HEAD)                 # remember for rollback
gh auth status 2>&1 | rg 'Logged in'           # gh ready (account chelout)
brew --repository chelout/tap                  # tap path exists locally
git log --oneline "HEAD..$LATEST" | head -40   # what is being pulled
# conflict preview WITHOUT mutating the working tree:
git merge-tree --write-tree --name-only HEAD "$LATEST" | tail -n +2
```

Report the conflicting files (if any) and the upstream highlights up front.

## Phase 2 — Merge

```bash
git merge --no-ff --no-commit "$LATEST"
```

- **Conflicts → STOP and resolve interactively.** Show
  `git status --short | rg '^(UU|AA|DD)'`.
  - Fork↔upstream conflicts are usually **additive**: both sides added different
    functions/tests in the same spot — keep **BOTH** (union). When git's
    interleaving of shared boilerplate (`#[test]`, `);`, `}`) makes hand-editing
    fragile, reconstruct from stages: `git show :2:<file>` (ours) and
    `git show :3:<file>` (theirs), splice `preamble + ours-block + theirs-block + tail`.
  - After resolving: `git add <files>`; verify no markers remain
    (`rg -c '^(<<<<<<<|=======|>>>>>>>)' <files>` → expect none); sanity-check brace
    balance for code files.
- Confirm the version came across:
  `rg '^version' Cargo.toml | head -1` → should be `"$VER"`.

## Phase 3 — Quality gate (local, mandatory)

```bash
TC="$HOME/.rustup/toolchains/stable-aarch64-apple-darwin/bin"
PATH="$TC:$PATH" "$TC/cargo" fmt --all
git diff --stat                                            # fmt should change nothing
PATH="$TC:$PATH" "$TC/cargo" clippy --all-targets 2>&1 | tail -15
PATH="$TC:$PATH" "$TC/cargo" test --all 2>&1 | tail -5
```

Require: fmt clean, **zero** clippy warnings, **0 failed** tests. If red → fix the
resolution (or STOP and offer rollback). For conflict-resolved files, optionally
run the two merged feature areas by name to confirm neither side was dropped.

## Phase 4 — Commit + tag (local)

```bash
git commit -m "Merge upstream/master into master ($LATEST)

<one or two lines: which conflicts were resolved and how (union, etc.)>

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

Pick the fork tag suffix `N` — **reset to 1 for a new base version**; bump only when
re-releasing the *same* base:

```bash
PREV=$(git tag -l "${LATEST}-fork.*" --sort=-v:refname | head -1)
# no PREV → N=1 ; else N = <suffix of PREV> + 1
FORKTAG="${LATEST}-fork.${N}"
git tag -a "$FORKTAG" -m "rtk ${VER}-fork.${N}

Sync with upstream ${LATEST} (<short highlights of upstream changes>).
Fork features retained: smart grep/rg rewrite + full-command tracking in gain --history."
```

## 🚦 Confirmation gate (the only one)

Summarize for the user and **wait for explicit confirmation**:

- the merge commit + `$FORKTAG`,
- the upstream highlights pulled,
- the outward-facing actions about to run: push `master` + `$FORKTAG` → `chelout/rtk`;
  GitHub release; tap `Formula/rtk.rb` bump + push.

Do not proceed to Phase 5 until the user says yes.

## Phase 5 — Publish (only after confirmation)

```bash
git push origin master
git push origin "$FORKTAG"

# prebuilt arm64 release asset (Homebrew installs this on macOS arm64)
PATH="$TC:$PATH" "$TC/cargo" build --release
tar -czf /tmp/rtk-aarch64-apple-darwin.tar.gz -C target/release rtk
PRE_SHA=$(shasum -a 256 /tmp/rtk-aarch64-apple-darwin.tar.gz | awk '{print $1}')

gh release create "$FORKTAG" /tmp/rtk-aarch64-apple-darwin.tar.gz \
  -R chelout/rtk --title "$FORKTAG" \
  --notes "<Fork release synced with upstream $LATEST. Upstream changes (bullets) +
           Fork-only features retained + Install: brew install chelout/tap/rtk>"

# source-archive checksum (GitHub generates this from the just-pushed tag)
SRC_SHA=$(curl -fsSL "https://github.com/chelout/rtk/archive/refs/tags/${FORKTAG}.tar.gz" \
  | shasum -a 256 | awk '{print $1}')
echo "prebuilt: $PRE_SHA"; echo "source:   $SRC_SHA"
```

Bump the tap formula `$(brew --repository chelout/tap)/Formula/rtk.rb` — **five edits**:

1. source `url` → `https://github.com/chelout/rtk/archive/refs/tags/${FORKTAG}.tar.gz`
2. `version "${VER}-fork.${N}"`
3. source `sha256` → `$SRC_SHA`
4. prebuilt `url` (inside `on_macos do … on_arm do`) →
   `https://github.com/chelout/rtk/releases/download/${FORKTAG}/rtk-aarch64-apple-darwin.tar.gz`
5. prebuilt `sha256` → `$PRE_SHA`

Validate, then commit + push the tap (the tap is a separate git repo on branch `main`):

```bash
brew style chelout/tap/rtk && brew audit chelout/tap/rtk && brew fetch chelout/tap/rtk
TAP="$(brew --repository chelout/tap)"
git -C "$TAP" add Formula/rtk.rb
git -C "$TAP" commit -m "rtk ${VER}-fork.${N}

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
git -C "$TAP" push origin main
```

`brew fetch` downloads the arm64 asset and verifies its sha256 against the formula —
treat a mismatch as a hard failure (wrong checksum or asset not yet propagated).

## Phase 6 — Verify + report

```bash
brew update >/dev/null && brew upgrade chelout/tap/rtk 2>&1 | tail -5
hash -r && rtk --version                          # expect $VER
( cd /tmp && rtk rewrite 'grep -rn foo src/' )    # fork feature → "rtk grep foo src/"
                                                  #   (exit 3 on success is expected)
```

Final report (table): merge commit, fork tag, GitHub release URL, tap commit,
installed version.

---

## Rollback

- Mid-merge, no commit yet: `git merge --abort`.
- After the merge commit but before any push: `git reset --hard "$ROLLBACK"`.
- After pushing master/tag: do not force-push silently — surface the situation to
  the user and let them choose (revert commit, delete+retag, etc.).

## Failure notes

- Dirty working tree at start → never begin; ask the user to stash/commit first.
- Quality gate red on a conflict resolution → the resolution is wrong; re-resolve,
  don't relax the gate.
- A red fork **CD** run after pushing is expected and benign (memory `fork-cd-fails`).
