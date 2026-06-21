---
name: update-from-upstream
description: >-
  Sync this fork (origin = talbor49/herdr) with the original repo
  (upstream = ogulcancelik/herdr) and merge the new upstream commits into the
  current working branch. Use this whenever the user wants to "update from
  upstream", "update from the original", "sync the fork", "pull in upstream
  changes", "merge upstream into my branch", "catch up with ogulcancelik", or
  asks to bring the latest herdr changes into their branch — even if they don't
  say the word "upstream". It walks the fetch → fast-forward master → merge →
  resolve conflicts (preserving local features) → validate → commit flow, and
  knows the conflict patterns and gotchas specific to this fork.
---

# Update from upstream

Bring the latest `ogulcancelik/herdr` (upstream) commits into this fork and merge
them into the user's working branch, **without losing local features or
uncommitted work**.

This is a fork-sync workflow, *not* the contribution flow in the root `CLAUDE.md`
(that one is about landing task branches back onto upstream). Here the data flows
the other way: upstream → fork.

## Topology (verify first, don't assume)

```bash
git remote -v
```

Expected:
- `origin`   = `git@github.com:talbor49/herdr.git` — the fork (push target; may need SSH set up)
- `upstream` = `https://github.com/ogulcancelik/herdr.git` — the original (read-only source)

The default branch is **`master`** (not `main`). The user's working branch is
whatever is currently checked out (e.g. `tal-changes`). Confirm with `git branch`.

## Workflow

### 1. Fetch and assess divergence

```bash
git fetch upstream
git rev-list --left-right --count master...upstream/master      # local-master-ahead  upstream-ahead
git rev-list --left-right --count HEAD...upstream/master         # your-branch-ahead   upstream-ahead
```

If `master` is `0` ahead, it's a clean tracking branch and can be fast-forwarded.
If it's ahead, there are local commits on `master` — stop and ask; don't force-move it.

### 2. Protect uncommitted work — this is non-negotiable

Before merging, the working tree **must be clean**:

```bash
git status
```

Per the user's standing instruction, never revert, stash-away, or clobber
uncommitted local changes you didn't make. If the tree is dirty and the dirty
files overlap with upstream's changes (check
`git diff --name-only master upstream/master`), a merge will abort anyway. **Ask
the user to commit/stash first, or confirm how to handle it.** Don't decide for them.

### 3. Fast-forward `master` to `upstream/master`

Do this without checking out `master` (you're on the working branch). It's only
safe when `master` is an ancestor of `upstream/master`:

```bash
git merge-base --is-ancestor master upstream/master && echo "safe" || echo "STOP: master has local commits"
git branch -f master upstream/master
```

### 4. Merge upstream into the working branch

Use diff3 conflict style so you can see the merge **base**, not just the two
sides — this is essential for understanding intent (configure once:
`git config merge.conflictStyle zdiff3`).

```bash
git merge upstream/master --no-edit
```

### 5. Resolve conflicts (see "Conflict patterns" below)

```bash
git diff --name-only --diff-filter=U     # files still conflicted
grep -rn -E '^(<<<<<<<|\|\|\|\|\|\|\||=======$|>>>>>>>)' src/ tests/   # find every marker
```

Resolve, then `git add <file>` each one. Verify no markers survive before building.

### 6. Validate

```bash
cargo check --all-targets          # fast compile gate — catches missing struct fields, signature drift
just test                          # cargo nextest + maintenance scripts
# if nextest isn't installed: cargo test --bin herdr --locked
#   plus: python3 -m unittest scripts.test_agent_detection_manifest_check scripts.test_changelog scripts.test_preview scripts.test_vendor_libghostty_vt
cargo fmt                          # then re-run; just check's fmt step is strict
```

On clippy: `just check` runs `clippy --all-targets -- -D warnings`, which is
stricter than upstream's CI. If you hit a wall of `uninlined_format_args`/style
errors, check whether the flagged files are **byte-identical to upstream**
(`git diff upstream/master -- <file>` is empty). If so they pre-date the merge —
don't churn unrelated upstream files to satisfy a stricter local clippy. Only fix
lints in code the merge actually touched, and say so to the user.

### 7. Commit (propose the message first)

Per `CLAUDE.md`, propose the commit message and get alignment before committing.
Don't push unless the user explicitly asks — and confirm whether they want
`master` and/or the working branch pushed (origin may need SSH auth fixed first).

One-line, lowercase, conventional-ish, describing what was integrated and what
local features were preserved, e.g.:

```
merge upstream/master: adopt <upstream change>; keep <local features>
```

## Conflict patterns (the ones that actually come up here)

Read the diff3 **base** section (between `|||||||` and `=======`) to know what each
side started from. Most conflicts here are around worktrees, the agent panel, and
sidebar/dialog UI — areas both sides iterate on.

- **Mechanical change on both sides** (e.g. upstream boxed an enum variant
  `Foo(Box<T>)` while you added a sibling variant) → **combine both**. Then grep
  every construction/match site and make them consistent (the compiler will list
  them).

- **Signature drift** (you added a parameter to a shared fn; upstream added new
  callers using the old arity) → keep your signature, **adapt upstream's new
  callers** to it.

- **Upstream moved/deleted code you patched** (e.g. upstream deleted an in-place
  handler that moved to a new file like `deferred.rs`, while you'd improved the
  in-place version) → take upstream's deletion (`git checkout --theirs -- <file>`
  is fine *only* if your sole change to that file was to the deleted code — verify
  with `git diff <merge-base>..HEAD -- <file>`), then **port your improvement into
  the new location**.

- **Add/add tests at the same spot** (base section empty, both sides added tests)
  → **keep both sets**, adapting any calls to the merged signatures.

- **Feature-level conflict** — upstream *removed or replaced* a feature you built
  (e.g. upstream replaced your `agent_panel_scope` toggle with an
  `agent_panel_sort` mode). The auto-merge will silently take upstream's deletion
  in non-conflicted files, so the feature half-survives and won't compile. **Stop
  and ask the user** which design wins, and which specific local sub-features they
  want preserved. Don't silently pick — this is a product decision, and the root
  `CLAUDE.md` says to ask on design forks.

- **Cross-file ripple**: an upstream struct gains a field → every construction
  site must set it; a renamed/removed field → every reader breaks. After resolving
  the visible conflicts, `cargo check` is what surfaces the rest. Re-grep for the
  removed symbol across `src/` to be sure nothing dangles.

## Gotchas

- **Don't diff against `master` after step 3.** Once you fast-forward `master` to
  `upstream/master`, `git diff master..HEAD` compares upstream to your branch, not
  your local changes. To see what *you* changed, diff against the **merge base**:
  `git merge-base HEAD upstream/master` (or use the `|||||||` base label git prints
  in the conflict, e.g. `git diff <base-sha>..HEAD -- <file>`).

- **Watch for a half-applied feature.** If a symbol compiles in one file but is
  "undeclared" elsewhere, the merge probably took upstream's deletion in the
  auto-merged files while a conflict hunk kept your version. Decide the feature's
  fate (step "Feature-level conflict") and make all files consistent.

- **Running the dev build from inside herdr**: clear inherited sockets so the debug
  binary talks to `herdr-dev`:
  `env -u HERDR_SOCKET_PATH -u HERDR_CLIENT_SOCKET_PATH cargo run -- <command>`.

- **Stray files**: don't `touch` source files to bust clippy's cache — it creates
  untracked files (e.g. an empty `src/lib.rs`) that can sneak into the commit.
  Use `cargo clean -p herdr` or just accept the cache if you only need a summary.
