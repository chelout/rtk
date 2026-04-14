# Design: Smart Grep Rewrite in registry.rs

**Date**: 2026-04-14
**Status**: Approved
**Problem**: 311 out of ~791 grep commands (39%) fail Clap parsing and fall back to raw grep with 0% token savings

## Problem Statement

RTK's hook rewrites `grep -rn pattern path` to `rtk grep -rn pattern path` via naive prefix replacement. But RTK's Clap definition for `Grep` doesn't recognize common grep flags (`-r`, `-A`, `-B`, `-E`, `-i`, `-o`, `-l`), causing parse failures. The fallback runs the original grep successfully, but loses all filtering/compression opportunity.

### Error breakdown from DB (311 total failures)

| Error | Count | % |
|-------|-------|---|
| `-r` unknown flag | 264 | 85% |
| `-B` unknown flag | 21 | 7% |
| `-A` unknown flag | 17 | 5% |
| `-E` unknown flag | 4 | 1.3% |
| `-o` unknown flag | 2 | 0.6% |
| `-i` unknown flag | 1 | 0.3% |
| `-l` conflicts with `--max-len` | 2 | 0.6% |

## Solution: Approach A — Smart Rewrite in registry.rs

Add a dedicated `rewrite_grep()` function in `registry.rs` that parses the grep command, classifies flags, and constructs a valid `rtk grep` invocation. Called from `rewrite_segment()` before the generic prefix-swap logic.

### Placement in architecture

```
rewrite_segment()
  ├── if rule == "rtk grep"      → rewrite_grep()       ← NEW
  ├── if golangci-lint            → parse_golangci_..()  ← existing
  ├── if rule == "rtk gh" + json  → return None          ← existing
  └── generic prefix swap                                ← existing fallback
```

**Files changed**: Only `src/discover/registry.rs`. No changes to `main.rs` (Clap), `grep_cmd.rs`, or `lexer.rs`.

## Algorithm

### Input

Raw command string (after env-prefix and redirect-suffix stripping):
```
grep -rn -A 3 GetProduct\|GetWidgetID /path --include=*.go
```

### Step 1: Strip grep/rg prefix

Using existing `strip_word_prefix()`:
```
-rn -A 3 GetProduct\|GetWidgetID /path --include=*.go
```

### Step 2: shell_split() into tokens

Using existing `shell_split()` from `lexer.rs`:
```
["-rn", "-A", "3", "GetProduct|GetWidgetID", "/path", "--include=*.go"]
```

Note: `shell_split` removes backslash escaping (`\|` becomes `|`) and strips quotes.

### Step 3: Classify each token

Iterate tokens with a state machine:

- **Flags to strip** (no-op for rg): remove silently
- **Flags to pass through**: collect into `extra_flags` vec
- **Skip-rewrite flags**: return `None` immediately
- **First positional**: pattern
- **Subsequent positionals**: paths
- **Value flags**: consume next token as the value

### Step 4: Construct result

```
rtk grep <quoted-pattern> <first-path> [-- <extra-flags> <extra-paths>]
```

The `--` separator tells Clap to put everything after it into `extra_args`, bypassing flag validation.

## Flag Classification

### Strip (no-op for rg)

| Flag | Reason |
|------|--------|
| `-r`, `-R`, `--recursive` | rg is recursive by default |
| `-n`, `--line-number` | rtk always enables line numbers |
| `-E`, `--extended-regexp` | rg uses Perl regex (superset of ERE) |
| `-G`, `--basic-regexp` | BRE `\|` → `\|` already handled in grep_cmd.rs |
| `-P`, `--perl-regexp` | rg default |
| `-H`, `--with-filename` | rg default |
| `--color=...`, `--colour=...` | rtk strips ANSI anyway |

### Pass through (into extra_args via --)

| Flag | Notes |
|------|-------|
| `-i`, `--ignore-case` | Direct rg equivalent |
| `-w`, `--word-regexp` | Direct rg equivalent |
| `-v`, `--invert-match` | Direct rg equivalent |
| `-o`, `--only-matching` | Direct rg equivalent |
| `-x`, `--line-regexp` | Direct rg equivalent |
| `-A NUM`, `--after-context=NUM` | Direct rg equivalent |
| `-B NUM`, `--before-context=NUM` | Direct rg equivalent |
| `-C NUM`, `--context=NUM` | Direct rg equivalent |
| `--include=GLOB` | Translate to `--glob=GLOB` for rg |
| `--exclude=GLOB` | Translate to `--glob=!GLOB` for rg |

### Skip rewrite (return None — let raw grep handle it)

| Flag | Reason |
|------|--------|
| `-l`, `--files-with-matches` | Conflicts with RTK's `-l` (max_len); different output format |
| `-L`, `--files-without-match` | Different output format |
| `-c`, `--count` | Conflicts with RTK's `-c` (context_only); different output format |
| `-q`, `--quiet`, `--silent` | No output to filter |
| `-f FILE`, `--file=FILE` | Pattern from file — complex, rare from Claude Code |
| `-Z`, `--null` | Machine-readable output |

### Value-taking short flags

These consume the next token as their value: `-A`, `-B`, `-C`, `-e`, `-f`, `-m`

### Special: `-e` flag (explicit pattern)

`grep -e pattern1 -e pattern2 path` uses `-e` to explicitly mark patterns. Handling:
- Single `-e pattern`: use as the pattern (equivalent to `grep pattern`)
- Multiple `-e` patterns: skip rewrite (rare from Claude Code, complex to merge)

### Special: `-m` / `--max-count`

Conflicts with RTK's `-m` (max results shown). Skip rewrite when present.

### Unknown flags

Any flag not listed above: **pass through** into extra_args. This makes the function forward-compatible — new grep/rg flags won't cause failures, they'll just be forwarded to rg.

## Combined Flags

POSIX rule: in a combined flag string, once a value-taking flag is encountered, the rest of the string is its value.

| Input | Parsed as | Result |
|-------|-----------|--------|
| `-rn` | `r`(strip) + `n`(strip) | nothing |
| `-rin` | `r`(strip) + `i`(pass) + `n`(strip) | `-i` |
| `-rnA3` | `r`(strip) + `n`(strip) + `A`(value=`3`) | `-A 3` |
| `-A3` | `A`(value=`3`) | `-A 3` |
| `-Arn` | `A`(value=`rn`) | `-A rn` |

## Multiple Paths

```
grep -rn pattern path1 path2 path3
→ rtk grep 'pattern' path1 -- path2 path3
```

First path → Clap `path` positional. Extra paths → `extra_args` via `--` → appended to rg command in `grep_cmd.rs` → rg accepts them as additional search paths.

## Re-quoting (shell_quote helper)

Since `shell_split()` removes quotes and escaping, the output must be re-quoted to prevent shell interpretation:

```rust
fn shell_quote(s: &str) -> String {
    if s.chars().all(|c| c.is_ascii_alphanumeric()
        || "-._/:=@%+\\".contains(c)) {
        return s.to_string();
    }
    format!("'{}'", s.replace('\'', "'\\''"))
}
```

Examples:
- `pattern` → `pattern`
- `GetProduct|GetWidgetID` → `'GetProduct|GetWidgetID'`
- `/simple/path` → `/simple/path`
- `it's` → `'it'\''s'`

## Real-World Examples (from DB)

| Input (from parse_failures) | Output |
|---|---|
| `grep -rn data\.SDKPartnerID\|data\.WidgetID internal/services/kyc/ internal/rest/` | `rtk grep 'data\.SDKPartnerID\|data\.WidgetID' internal/services/kyc/ -- internal/rest/` |
| `grep -rn BuildMetadata\b /path/ --include=*.go` | `rtk grep 'BuildMetadata\b' /path/ -- --glob=*.go` |
| `grep -l testify\|require\. /path/` | `None` (skip — `-l` incompatible) |
| `grep -E '^  (test\|lint\|go)' /path/taskfile.yml` | `rtk grep '^  (test\|lint\|go)' /path/taskfile.yml` |
| `grep -rn -B 5 pattern path` | `rtk grep pattern path -- -B 5` |
| `grep -rn 'Product\b\|Scenario\b' /path/file.go` | `rtk grep 'Product\b\|Scenario\b' /path/file.go` |

## Testing Strategy

1. **Unit tests for `rewrite_grep()`**: Cover all flag categories (strip, pass-through, skip), combined flags, multiple paths, quoting edge cases
2. **Unit tests for `shell_quote()`**: Safe chars, special chars, single quotes, empty string
3. **Regression tests from DB**: Use actual failing commands from `parse_failures` table as test fixtures
4. **Integration**: Verify the rewritten commands parse successfully through Clap

## Scope Boundaries

- **In scope**: `rewrite_grep()` function, `shell_quote()` helper, tests
- **Out of scope**: Changes to `main.rs` (Clap definition), `grep_cmd.rs` (filter logic), `lexer.rs`
- **Out of scope**: Fixing the dead `-r` stripping code in `grep_cmd.rs:42` (still useful as defense-in-depth)
