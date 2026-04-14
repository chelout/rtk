# Smart Grep Rewrite — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fix 39% grep command failure rate by adding a smart `rewrite_grep()` function in `registry.rs` that translates grep flags into valid `rtk grep` invocations.

**Architecture:** A new `rewrite_grep()` function is called from `rewrite_segment()` when the matched rule is `"rtk grep"`. It parses the grep command via `shell_split()`, classifies flags (strip / pass-through / skip-rewrite), and constructs a valid `rtk grep <pattern> <path> [-- <extra_args>]` command. A `shell_quote()` helper re-quotes pattern/path tokens for safe shell execution.

**Tech Stack:** Rust, no new dependencies. Uses existing `shell_split()` from `src/discover/lexer.rs`.

**Spec:** `docs/superpowers/specs/2026-04-14-grep-rewrite-design.md`

---

## File Map

| File | Action | Responsibility |
|------|--------|----------------|
| `src/discover/registry.rs` | Modify | Add `shell_quote()`, `rewrite_grep()`, hook into `rewrite_segment()`, add tests |

No other files are created or modified.

---

### Task 1: Add `shell_quote()` helper with tests

**Files:**
- Modify: `src/discover/registry.rs` (add function + tests at end of `mod tests`)

- [ ] **Step 1: Write the failing tests for `shell_quote()`**

Add these tests at the end of the existing `mod tests` block in `registry.rs` (before the final closing `}`), around line 2726:

```rust
    // --- shell_quote ---

    #[test]
    fn test_shell_quote_safe_chars() {
        assert_eq!(shell_quote("pattern"), "pattern");
        assert_eq!(shell_quote("src/main.rs"), "src/main.rs");
        assert_eq!(shell_quote("/Users/chelout/code"), "/Users/chelout/code");
        assert_eq!(shell_quote("-A"), "-A");
        assert_eq!(shell_quote("3"), "3");
    }

    #[test]
    fn test_shell_quote_glob() {
        // * triggers quoting (prevents shell glob expansion)
        assert_eq!(shell_quote("--glob=*.go"), "'--glob=*.go'");
    }

    #[test]
    fn test_shell_quote_pipe() {
        assert_eq!(shell_quote("GetProduct|GetWidgetID"), "'GetProduct|GetWidgetID'");
    }

    #[test]
    fn test_shell_quote_spaces() {
        assert_eq!(shell_quote("hello world"), "'hello world'");
    }

    #[test]
    fn test_shell_quote_single_quote() {
        assert_eq!(shell_quote("it's"), "'it'\\''s'");
    }

    #[test]
    fn test_shell_quote_parens() {
        assert_eq!(shell_quote("^  (test|lint)"), "'^  (test|lint)'");
    }

    #[test]
    fn test_shell_quote_empty() {
        assert_eq!(shell_quote(""), "''");
    }

    #[test]
    fn test_shell_quote_backslash() {
        // Backslash is NOT safe — must be quoted to prevent shell interpretation
        assert_eq!(shell_quote("data\\.Field"), "'data\\.Field'");
        assert_eq!(shell_quote("BuildMetadata\\b"), "'BuildMetadata\\b'");
    }
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cargo test shell_quote -- --nocapture 2>&1 | head -20`
Expected: compilation error — `shell_quote` function not defined.

- [ ] **Step 3: Implement `shell_quote()`**

Add this function in `registry.rs`, right before the `#[cfg(test)]` block (around line 711):

```rust
/// Re-quote a token for safe shell execution.
/// Tokens that contain only shell-safe characters are returned as-is.
/// All others are wrapped in single quotes with internal `'` escaped.
fn shell_quote(s: &str) -> String {
    if s.is_empty() {
        return "''".to_string();
    }
    if s.chars()
        .all(|c| c.is_ascii_alphanumeric() || "-._/:=@%+".contains(c))
    {
        return s.to_string();
    }
    format!("'{}'", s.replace('\'', "'\\''"))
}
```

Note: backslash (`\`) is intentionally NOT in the safe set. In shell, `\b` is interpreted as just `b` (backslash escapes the next char). If a pattern like `BuildMetadata\b` was originally single-quoted, `shell_split` preserves the backslash, and `shell_quote` must re-quote it to prevent shell interpretation.

- [ ] **Step 4: Run tests to verify they pass**

Run: `cargo test shell_quote -- --nocapture`
Expected: all 7 tests PASS.

- [ ] **Step 5: Run full test suite**

Run: `cargo test --all 2>&1 | tail -5`
Expected: all existing tests still pass.

- [ ] **Step 6: Commit**

```bash
git add src/discover/registry.rs
git commit -m "feat(grep): add shell_quote() helper for safe re-quoting"
```

---

### Task 2: Add `rewrite_grep()` function with core flag parsing

**Files:**
- Modify: `src/discover/registry.rs` (add function after `shell_quote`, add tests)

- [ ] **Step 1: Write failing tests for basic grep rewrite**

Add these tests at the end of `mod tests` in `registry.rs`:

```rust
    // --- rewrite_grep ---

    #[test]
    fn test_rewrite_grep_strip_r_flag() {
        assert_eq!(
            rewrite_command("grep -rn pattern /path", &[]),
            Some("rtk grep pattern /path".into())
        );
    }

    #[test]
    fn test_rewrite_grep_strip_rn_combined() {
        assert_eq!(
            rewrite_command("grep -rn GetProduct src/", &[]),
            Some("rtk grep GetProduct src/".into())
        );
    }

    #[test]
    fn test_rewrite_grep_strip_E_flag() {
        assert_eq!(
            rewrite_command("grep -E pattern file.txt", &[]),
            Some("rtk grep pattern file.txt".into())
        );
    }

    #[test]
    fn test_rewrite_grep_passthrough_i() {
        assert_eq!(
            rewrite_command("grep -ri pattern /path", &[]),
            Some("rtk grep pattern /path -- -i".into())
        );
    }

    #[test]
    fn test_rewrite_grep_passthrough_A_with_value() {
        assert_eq!(
            rewrite_command("grep -rn -A 3 pattern /path", &[]),
            Some("rtk grep pattern /path -- -A 3".into())
        );
    }

    #[test]
    fn test_rewrite_grep_passthrough_B_with_value() {
        assert_eq!(
            rewrite_command("grep -B 5 pattern /path", &[]),
            Some("rtk grep pattern /path -- -B 5".into())
        );
    }

    #[test]
    fn test_rewrite_grep_skip_l_flag() {
        // -l conflicts with RTK's --max-len; different output format
        assert_eq!(rewrite_command("grep -l pattern /path", &[]), None);
    }

    #[test]
    fn test_rewrite_grep_skip_c_flag() {
        // -c conflicts with RTK's --context-only; different output format
        assert_eq!(rewrite_command("grep -c pattern /path", &[]), None);
    }

    #[test]
    fn test_rewrite_grep_skip_q_flag() {
        assert_eq!(rewrite_command("grep -q pattern /path", &[]), None);
    }

    #[test]
    fn test_rewrite_grep_default_path() {
        assert_eq!(
            rewrite_command("grep -rn pattern", &[]),
            Some("rtk grep pattern .".into())
        );
    }

    #[test]
    fn test_rewrite_grep_bare_pattern_only() {
        assert_eq!(
            rewrite_command("grep pattern", &[]),
            Some("rtk grep pattern .".into())
        );
    }
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cargo test test_rewrite_grep -- --nocapture 2>&1 | head -30`
Expected: tests FAIL — `rewrite_grep` not yet called, the generic rewrite produces wrong output (e.g., `rtk grep -rn pattern /path` instead of `rtk grep pattern /path`).

- [ ] **Step 3: Implement `rewrite_grep()`**

Add this function in `registry.rs`, right after the `shell_quote()` function (before the `#[cfg(test)]` block). Also add `shell_split` to the import at the top of the file.

First, update the import on line 6:

```rust
use super::lexer::{shell_split, tokenize, TokenKind};
```

Then add the function:

```rust
/// Smart rewrite for grep/rg commands.
/// Parses grep flags, strips no-ops (e.g. -r, -n), passes through rg-compatible
/// flags via `--`, and skips rewrite for incompatible modes (e.g. -l, -c).
/// Returns `Some(rewritten)` on success, `None` to skip rewrite.
fn rewrite_grep(cmd_clean: &str, env_prefix: &str, redirect_suffix: &str) -> Option<String> {
    // Strip "grep" or "rg" prefix to get the arguments portion
    let args_str = if let Some(rest) = strip_word_prefix(cmd_clean, "grep") {
        rest
    } else if let Some(rest) = strip_word_prefix(cmd_clean, "rg") {
        rest
    } else {
        return None;
    };

    // Bare "grep" / "rg" with no arguments — skip
    if args_str.is_empty() {
        return None;
    }

    let tokens = shell_split(args_str);
    if tokens.is_empty() {
        return None;
    }

    // Flags that mean "strip" (no-op for rg)
    const STRIP_SHORT: &[char] = &['r', 'R', 'n', 'E', 'G', 'P', 'H'];
    // Flags that mean "skip rewrite entirely"
    const SKIP_SHORT: &[char] = &['l', 'L', 'c', 'q'];
    // Short flags that take a required value
    const VALUE_SHORT: &[char] = &['A', 'B', 'C', 'e', 'f', 'm'];

    let mut extra_flags: Vec<String> = Vec::new();
    let mut pattern: Option<String> = None;
    let mut paths: Vec<String> = Vec::new();
    let mut explicit_patterns: Vec<String> = Vec::new(); // from -e
    let mut past_double_dash = false;

    let mut i = 0;
    while i < tokens.len() {
        let tok = &tokens[i];

        // After --, everything is positional
        if past_double_dash {
            if pattern.is_none() {
                pattern = Some(tok.clone());
            } else {
                paths.push(tok.clone());
            }
            i += 1;
            continue;
        }

        if tok == "--" {
            past_double_dash = true;
            i += 1;
            continue;
        }

        // Long options
        if tok.starts_with("--") {
            match tok.as_str() {
                "--recursive" | "--line-number" | "--extended-regexp"
                | "--basic-regexp" | "--perl-regexp" | "--with-filename" => {
                    // strip — no-op for rg
                }
                "--files-with-matches" | "--files-without-match"
                | "--count" | "--quiet" | "--silent" => {
                    return None; // skip rewrite
                }
                "--max-count" => {
                    return None; // conflicts with RTK -m
                }
                _ if tok.starts_with("--color") || tok.starts_with("--colour") => {
                    // strip — rtk strips ANSI
                }
                _ if tok.starts_with("--include=") => {
                    let glob = &tok["--include=".len()..];
                    extra_flags.push(format!("--glob={}", glob));
                }
                _ if tok.starts_with("--exclude=") => {
                    let glob = &tok["--exclude=".len()..];
                    extra_flags.push(format!("--glob=!{}", glob));
                }
                _ if tok.starts_with("--file=") => {
                    return None; // -f FILE — skip rewrite
                }
                _ if tok.starts_with("--max-count=") => {
                    return None; // conflicts with RTK -m
                }
                _ => {
                    // Unknown long flag — pass through for forward compatibility
                    extra_flags.push(tok.clone());
                }
            }
            i += 1;
            continue;
        }

        // Short options (single or combined)
        if tok.starts_with('-') && tok.len() > 1 && tok.as_bytes()[1] != b'.' {
            let chars: Vec<char> = tok[1..].chars().collect();
            let mut j = 0;
            while j < chars.len() {
                let c = chars[j];

                if SKIP_SHORT.contains(&c) {
                    return None;
                }

                if STRIP_SHORT.contains(&c) {
                    j += 1;
                    continue;
                }

                if VALUE_SHORT.contains(&c) {
                    // Value is the rest of this combined flag, or the next token
                    let value = if j + 1 < chars.len() {
                        let v: String = chars[j + 1..].iter().collect();
                        Some(v)
                    } else if i + 1 < tokens.len() {
                        i += 1;
                        Some(tokens[i].clone())
                    } else {
                        None
                    };

                    if c == 'e' {
                        // -e PATTERN: explicit pattern
                        if let Some(v) = value {
                            explicit_patterns.push(v);
                        }
                    } else if c == 'f' || c == 'm' {
                        // -f FILE or -m NUM: skip rewrite
                        return None;
                    } else {
                        // -A, -B, -C: pass through with value
                        extra_flags.push(format!("-{}", c));
                        if let Some(v) = value {
                            extra_flags.push(v);
                        }
                    }
                    break; // rest consumed as value
                }

                // Unknown short flag — pass through
                extra_flags.push(format!("-{}", c));
                j += 1;
            }
            i += 1;
            continue;
        }

        // Positional argument
        if pattern.is_none() && explicit_patterns.is_empty() {
            pattern = Some(tok.clone());
        } else {
            paths.push(tok.clone());
        }
        i += 1;
    }

    // Multiple -e patterns: skip rewrite (complex to merge)
    if explicit_patterns.len() > 1 {
        return None;
    }

    // Single -e pattern takes precedence over positional
    if let Some(ep) = explicit_patterns.into_iter().next() {
        if let Some(positional) = pattern.take() {
            // The positional was actually a path, not a pattern
            paths.insert(0, positional);
        }
        pattern = Some(ep);
    }

    let pattern = pattern?; // No pattern at all — skip rewrite
    let first_path = if paths.is_empty() {
        ".".to_string()
    } else {
        paths.remove(0)
    };

    // Build the rewritten command
    let quoted_pattern = shell_quote(&pattern);
    let quoted_path = shell_quote(&first_path);

    let has_extras = !extra_flags.is_empty() || !paths.is_empty();

    let mut result = format!("{}rtk grep {} {}", env_prefix, quoted_pattern, quoted_path);

    if has_extras {
        result.push_str(" --");
        for flag in &extra_flags {
            result.push(' ');
            result.push_str(&shell_quote(flag));
        }
        for path in &paths {
            result.push(' ');
            result.push_str(&shell_quote(path));
        }
    }

    result.push_str(redirect_suffix);
    Some(result)
}
```

- [ ] **Step 4: Hook `rewrite_grep()` into `rewrite_segment()`**

In `rewrite_segment()`, add the grep special case right before the golangci-lint check (around line 658). Insert this block:

```rust
    // Smart rewrite for grep/rg — translate grep flags to rtk grep format
    if rule.rtk_cmd == "rtk grep" {
        return rewrite_grep(cmd_clean, env_prefix, redirect_suffix);
    }
```

The exact insertion point is between the `RTK_DISABLED` check (line 656) and the golangci-lint check (line 658). After the edit, the code should look like:

```rust
    // ... RTK_DISABLED check above ...

    // Smart rewrite for grep/rg — translate grep flags to rtk grep format
    if rule.rtk_cmd == "rtk grep" {
        return rewrite_grep(cmd_clean, env_prefix, redirect_suffix);
    }

    if let Some(parts) = parse_golangci_run_parts(cmd_clean) {
    // ... existing golangci-lint code ...
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `cargo test test_rewrite_grep -- --nocapture`
Expected: all 12 tests PASS.

- [ ] **Step 6: Run full test suite to check for regressions**

Run: `cargo fmt --all && cargo clippy --all-targets && cargo test --all`
Expected: zero warnings, all tests pass. The existing `test_rewrite_rg_pattern` test may need updating — it currently expects `rtk grep "fn main"` but the new rewrite will produce `rtk grep 'fn main'` (shell_quote uses single quotes for strings with spaces). If it fails, update the expected value.

- [ ] **Step 7: Commit**

```bash
git add src/discover/registry.rs
git commit -m "feat(grep): add rewrite_grep() with flag parsing and strip/pass/skip logic"
```

---

### Task 3: Add tests for combined flags, quoting, and edge cases

**Files:**
- Modify: `src/discover/registry.rs` (add more tests to `mod tests`)

- [ ] **Step 1: Add combined flag tests**

```rust
    #[test]
    fn test_rewrite_grep_combined_rin() {
        // -r strip, -i pass, -n strip
        assert_eq!(
            rewrite_command("grep -rin pattern /path", &[]),
            Some("rtk grep pattern /path -- -i".into())
        );
    }

    #[test]
    fn test_rewrite_grep_combined_rnA_with_inline_value() {
        // -r strip, -n strip, -A takes "3" as value
        assert_eq!(
            rewrite_command("grep -rnA3 pattern /path", &[]),
            Some("rtk grep pattern /path -- -A 3".into())
        );
    }

    #[test]
    fn test_rewrite_grep_combined_A_inline_value() {
        assert_eq!(
            rewrite_command("grep -A3 pattern /path", &[]),
            Some("rtk grep pattern /path -- -A 3".into())
        );
    }
```

- [ ] **Step 2: Add quoting tests**

```rust
    #[test]
    fn test_rewrite_grep_pipe_in_pattern_quoted() {
        // Unquoted \| gets shell_split into |, must be re-quoted
        assert_eq!(
            rewrite_command(r"grep -rn GetProduct\|GetWidgetID /path", &[]),
            Some("rtk grep 'GetProduct|GetWidgetID' /path".into())
        );
    }

    #[test]
    fn test_rewrite_grep_single_quoted_pattern() {
        assert_eq!(
            rewrite_command("grep -rn 'Product\\b|Scenario\\b' /path/file.go", &[]),
            Some("rtk grep 'Product\\b|Scenario\\b' /path/file.go".into())
        );
    }

    #[test]
    fn test_rewrite_grep_parens_in_pattern() {
        assert_eq!(
            rewrite_command("grep -E '^  (test|lint|go)' /path/taskfile.yml", &[]),
            Some("rtk grep '^  (test|lint|go)' /path/taskfile.yml".into())
        );
    }
```

- [ ] **Step 3: Add multiple paths tests**

```rust
    #[test]
    fn test_rewrite_grep_multiple_paths() {
        assert_eq!(
            rewrite_command("grep -rn pattern path1 path2 path3", &[]),
            Some("rtk grep pattern path1 -- path2 path3".into())
        );
    }

    #[test]
    fn test_rewrite_grep_multiple_paths_with_flags() {
        assert_eq!(
            rewrite_command("grep -rn -A 3 pattern path1 path2", &[]),
            Some("rtk grep pattern path1 -- -A 3 path2".into())
        );
    }
```

- [ ] **Step 4: Add --include/--exclude translation tests**

```rust
    #[test]
    fn test_rewrite_grep_include_to_glob() {
        assert_eq!(
            rewrite_command("grep -rn pattern /path --include=*.go", &[]),
            Some("rtk grep pattern /path -- '--glob=*.go'".into())
        );
    }

    #[test]
    fn test_rewrite_grep_exclude_to_glob_negated() {
        assert_eq!(
            rewrite_command("grep -rn pattern /path --exclude=*.tmp", &[]),
            Some("rtk grep pattern /path -- '--glob=!*.tmp'".into())
        );
    }
```

- [ ] **Step 5: Add -e flag and edge case tests**

```rust
    #[test]
    fn test_rewrite_grep_explicit_e_pattern() {
        assert_eq!(
            rewrite_command("grep -rn -e pattern /path", &[]),
            Some("rtk grep pattern /path".into())
        );
    }

    #[test]
    fn test_rewrite_grep_multiple_e_skip() {
        // Multiple -e patterns: skip rewrite
        assert_eq!(
            rewrite_command("grep -e pat1 -e pat2 /path", &[]),
            None
        );
    }

    #[test]
    fn test_rewrite_grep_skip_m_flag() {
        // -m conflicts with RTK's --max
        assert_eq!(rewrite_command("grep -m 5 pattern /path", &[]), None);
    }

    #[test]
    fn test_rewrite_grep_skip_f_flag() {
        assert_eq!(
            rewrite_command("grep -f patterns.txt /path", &[]),
            None
        );
    }

    #[test]
    fn test_rewrite_grep_unknown_flag_passthrough() {
        // Unknown flags go to extra_args for forward compatibility
        assert_eq!(
            rewrite_command("grep --some-future-flag pattern /path", &[]),
            Some("rtk grep pattern /path -- --some-future-flag".into())
        );
    }

    #[test]
    fn test_rewrite_grep_rg_prefix() {
        assert_eq!(
            rewrite_command("rg -i pattern /path", &[]),
            Some("rtk grep pattern /path -- -i".into())
        );
    }
```

- [ ] **Step 6: Run all tests**

Run: `cargo fmt --all && cargo clippy --all-targets && cargo test --all`
Expected: all tests pass, zero warnings.

- [ ] **Step 7: Commit**

```bash
git add src/discover/registry.rs
git commit -m "test(grep): add combined flags, quoting, multi-path, include/exclude tests"
```

---

### Task 4: Add regression tests from real DB failures

**Files:**
- Modify: `src/discover/registry.rs` (add tests from actual parse_failures data)

- [ ] **Step 1: Add regression tests matching real failing commands from the database**

These are exact commands from the `parse_failures` table that were failing before the fix:

```rust
    // --- grep rewrite: regression tests from parse_failures DB ---

    #[test]
    fn test_rewrite_grep_regression_rn_backslash_pipe_multi_path() {
        // parse_failures id 314: grep with \| alternation and 3 paths
        assert_eq!(
            rewrite_command(
                r"grep -rn data\.SDKPartnerID\|data\.WidgetID internal/services/kyc/ internal/rest/ internal/domain/kyc/",
                &[],
            ),
            Some("rtk grep 'data.SDKPartnerID|data.WidgetID' internal/services/kyc/ -- internal/rest/ internal/domain/kyc/".into())
        );
    }

    #[test]
    fn test_rewrite_grep_regression_include_glob() {
        // parse_failures id 311: grep with --include
        assert_eq!(
            rewrite_command(
                r"grep -rn BuildMetadata\b /Users/chelout/code/risk/ --include=*.go",
                &[],
            ),
            Some("rtk grep 'BuildMetadata\\b' /Users/chelout/code/risk/ -- --glob=*.go".into())
        );
    }

    #[test]
    fn test_rewrite_grep_regression_l_flag_skip() {
        // parse_failures id 297: -l conflicts with RTK
        assert_eq!(
            rewrite_command(
                r"grep -l testify\|require\. /Users/chelout/code/risk/internal/rest/validation/",
                &[],
            ),
            None
        );
    }

    #[test]
    fn test_rewrite_grep_regression_E_flag() {
        // parse_failures id 296: -E extended regex
        assert_eq!(
            rewrite_command(
                "grep -E '^  (test|lint|go)' /Users/chelout/code/risk/taskfile.yml",
                &[],
            ),
            Some("rtk grep '^  (test|lint|go)' /Users/chelout/code/risk/taskfile.yml".into())
        );
    }

    #[test]
    fn test_rewrite_grep_regression_B_flag() {
        // parse_failures: -B context lines
        assert_eq!(
            rewrite_command("grep -rn -B 5 pattern /path", &[]),
            Some("rtk grep pattern /path -- -B 5".into())
        );
    }

    #[test]
    fn test_rewrite_grep_regression_long_multi_path() {
        // parse_failures id 306: many individual file paths
        let cmd = "grep -rn 'package validation' /path/a.go /path/b.go /path/c.go";
        let result = rewrite_command(cmd, &[]);
        assert!(result.is_some());
        let r = result.unwrap();
        assert!(r.starts_with("rtk grep"));
        assert!(r.contains("-- /path/b.go /path/c.go"));
    }
```

- [ ] **Step 2: Run all tests**

Run: `cargo fmt --all && cargo clippy --all-targets && cargo test --all`
Expected: all tests pass. If any regression test fails, adjust the expected output to match what `rewrite_grep()` actually produces and verify it is a correct rewrite.

- [ ] **Step 3: Commit**

```bash
git add src/discover/registry.rs
git commit -m "test(grep): add regression tests from real parse_failures DB data"
```

---

### Task 5: Fix the existing `test_rewrite_rg_pattern` test if broken

**Files:**
- Modify: `src/discover/registry.rs` (update existing test if needed)

- [ ] **Step 1: Check if the existing rg rewrite test still passes**

Run: `cargo test test_rewrite_rg_pattern -- --nocapture`

The existing test at line 1239 expects:
```rust
assert_eq!(
    rewrite_command("rg \"fn main\"", &[]),
    Some("rtk grep \"fn main\"".into())
);
```

With the new `rewrite_grep()`, this command will be parsed as: pattern=`fn main` (quotes stripped by shell_split), path=`.` (default). The output will be `rtk grep 'fn main' .` instead of `rtk grep "fn main"`.

- [ ] **Step 2: Update the test if it fails**

If the test fails, update the expected value:

```rust
    #[test]
    fn test_rewrite_rg_pattern() {
        assert_eq!(
            rewrite_command("rg \"fn main\"", &[]),
            Some("rtk grep 'fn main' .".into())
        );
    }
```

- [ ] **Step 3: Run full quality gate**

Run: `cargo fmt --all && cargo clippy --all-targets && cargo test --all`
Expected: ALL tests pass, zero clippy warnings.

- [ ] **Step 4: Commit (only if changes were needed)**

```bash
git add src/discover/registry.rs
git commit -m "fix(grep): update existing rg rewrite test for new shell_quote behavior"
```

---

### Task 6: Final verification

- [ ] **Step 1: Run full pre-commit gate**

Run: `cargo fmt --all && cargo clippy --all-targets && cargo test --all`
Expected: clean pass.

- [ ] **Step 2: Build release binary**

Run: `cargo build --release`
Expected: success.

- [ ] **Step 3: Manual smoke test with real grep commands**

Run these commands to verify the rewrite produces valid output:

```bash
# Should produce a valid rtk grep command, not fallback
./target/release/rtk rewrite "grep -rn pattern src/"
# Expected output: rtk grep pattern src/

./target/release/rtk rewrite "grep -rn -A 3 pattern src/"
# Expected output: rtk grep pattern src/ -- -A 3

./target/release/rtk rewrite "grep -l pattern src/"
# Expected output: (empty, exit code 1 = no rewrite)

./target/release/rtk rewrite "grep -E 'test|lint' Makefile"
# Expected output: rtk grep 'test|lint' Makefile
```

- [ ] **Step 4: Performance check**

Run: `hyperfine './target/release/rtk rewrite "grep -rn pattern src/"' --warmup 3`
Expected: <10ms (rewrite is fast string manipulation).

- [ ] **Step 5: Commit any final adjustments**

If smoke tests revealed issues, fix and commit. Otherwise, no action needed.
