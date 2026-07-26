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
  resolve conflicts (preserving local features) → validate → commit → report
  what's new flow, and knows the conflict patterns and gotchas specific to this
  fork.
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

**Clean auto-merge is common** (e.g. the 23-commit sync where local and upstream
changes touched disjoint regions). When `git merge` reports **zero conflicts**, it
**auto-creates the merge commit with git's generic default message**
("Merge remote-tracking branch 'upstream/master'…"). Don't skip validation just
because there were no conflicts — a clean *textual* merge can still break the build
via cross-file ripple (see Conflict patterns). And plan to **amend that auto-commit
to a descriptive one-liner** (step 7) before pushing.

### 5. Resolve conflicts (see "Conflict patterns" below)

```bash
git diff --name-only --diff-filter=U     # files still conflicted
grep -rn -E '^(<<<<<<<|\|\|\|\|\|\|\||=======$|>>>>>>>)' src/ tests/   # find every marker
```

Resolve, then `git add <file>` each one. Verify no markers survive before building.

#### 5b. Audit for silently dropped local code — do not skip this

Conflict markers only appear where both sides touched the *same* lines. When upstream
rewrites a region that local merely added to, or deletes something local still depends
on, git resolves it silently and the local change vanishes with no marker and often no
compile error. This has really happened here: upstream deleted
`Tab::follow_cwd_for_pane` (its callers moved into `app::creation`), and the auto-merge
took that deletion even though a local commit had added a third caller in
`workspace.rs`.

Run this **before committing** (during the merge, so `HEAD` is still the pre-merge
local tip and `MERGE_HEAD` is upstream). It lists every line local added that is no
longer anywhere in the merged tree:

```bash
python3 - <<'PY'
import subprocess, collections
def sh(*a): return subprocess.run(["git",*a],capture_output=True,text=True).stdout.strip()
if sh("rev-parse","-q","--verify","MERGE_HEAD"):        # mid-merge, not yet committed
    base, local = sh("merge-base","HEAD","MERGE_HEAD"), "HEAD"
else:                                                   # merge already committed
    base, local = sh("merge-base","HEAD^1","HEAD^2"), "HEAD^1"
print(f"base={base} local={local}")
diff = sh("diff","-U0",base,local,"--","src/","tests/")
cur, added = None, collections.defaultdict(list)
for line in diff.splitlines():
    if line.startswith("+++ b/"): cur = line[6:]
    elif line.startswith("+") and cur and not line.startswith("+++"):
        s = line[1:].strip()
        if len(s) > 12 and s not in ("{","}"): added[cur].append(s)
total = 0
for f in sorted(added):
    try: body = open(f).read()
    except FileNotFoundError: print(f"\n### {f}  <FILE GONE>"); continue
    gone = [s for s in added[f] if s not in body]
    if not gone: continue
    print(f"\n### {f}  ({len(gone)} missing)")
    for s in gone[:15]: print("   -", s[:116])
    total += len(gone)
print(f"\nTOTAL missing local-added lines: {total}")
PY
```

Every hit is one of exactly two things, and you must account for each:

1. **A line you rewrote yourself while resolving** — renamed symbol, merged import list,
   reindented call. Expected; tick it off.
2. **A local change the merge silently dropped.** Fix it: find where upstream moved the
   functionality (`git show <upstream-commit>` for the commit that deleted it) and port
   the local change onto the new shape.

Don't just eyeball the count — walk the list. A tidy run is ~10-15 hits, all category 1.
If you can't explain a hit, it's category 2.

When porting into a new location, check the layering first: upstream often moves a helper
somewhere the local caller can't legally reach (e.g. `pub(super)` inside `app/` when the
caller lives in `workspace.rs`, which `app/` depends on). Inlining the body at the single
remaining call site beats widening upstream's visibility or reverting their deletion.

### 6. Validate

Run these in order — stop and fix at the first real failure:

```bash
cargo check --all-targets                        # fast compile gate — missing struct fields, signature drift
just test                                        # nextest + Python maintenance tests (+ bun tests, see below)
cargo fmt --check                                # just check's fmt step is strict; `cargo fmt` to fix, then re-check
cargo clippy --all-targets --locked -- -D warnings   # native strict lint
just windows-lint                                # Windows-target clippy; run when the merge touched src/platform/windows.rs or src/remote/
```

`just windows-lint` needs the `x86_64-pc-windows-msvc` target; the recipe
`rustup target add`s it automatically. On a fresh machine the first run downloads
the target and takes a while.

**The `bun` tooling gap (expected on this machine).** `just test` runs, after the
Rust + Python tests, `just integration-assets-test` and `just plugin-marketplace-test`,
which both shell out to `bun test`. `bun` is **not installed here** (node/npm are),
so those two steps fail with `sh: bun: command not found` / exit 127. That is a
**local tooling gap, not a merge regression** — report it as "not run", don't treat
it as a failure. The real signal from `just test` is the part that runs first:
`2595 tests run: … passed` (nextest) and the Python `unittest` block `OK`. If the
merge touched a bundled TS integration asset (e.g. `src/integration/assets/…`), say
so and note the TS test couldn't be exercised locally — the Rust
`src/integration/tests.rs` still gives real coverage.

If nextest is somehow unavailable, the fallback is
`cargo test --bin herdr --locked` plus the current Python maintenance suite:
`python3 -m unittest scripts.test_agent_detection_manifest_check scripts.test_changelog scripts.test_docs_translation_parity scripts.test_preview scripts.test_vendor_libghostty_vt scripts.test_vendor_portable_pty`
(the exact list lives in the `test:` recipe in `justfile` — read it rather than
trusting this from memory; it grows over time).

On clippy: `clippy --all-targets -- -D warnings` is stricter than upstream's CI. If
you hit a wall of `uninlined_format_args`/style errors, check whether the flagged
files are **byte-identical to upstream** (`git diff upstream/master -- <file>` is
empty). If so they pre-date the merge — don't churn unrelated upstream files to
satisfy a stricter local clippy. Only fix lints in code the merge actually touched,
and say so to the user.

### 7. Commit (propose the message first)

Per `CLAUDE.md`, propose the commit message and get alignment before committing.

If the merge **auto-committed** (clean, no conflicts), the commit already exists
with git's default message — **amend it** to the descriptive one-liner rather than
adding a second commit:

```bash
git commit --amend -m "merge upstream/master: <upstream highlights>; <what local work was preserved>"
```

If you resolved conflicts, `git add` the resolved files and `git commit` normally.

Either way the subject is one line, lowercase, conventional-ish, describing what was
integrated and what local features were preserved, e.g.:

```
merge upstream/master: adopt <upstream change>; keep <local features>
```

Don't push unless the user explicitly asks — and confirm whether they want `master`
and/or the working branch pushed (origin may need SSH auth fixed first). Push the
working branch with `git push origin <branch>`.

### 8. Install the updated binary

After validation passes and the merge is committed, reinstall so the local `herdr`
binary reflects the merged tree:

```bash
cargo install --path .
```

On this macOS 27 machine the vendored libghostty-vt build works through the `zig`
wrapper under `~/.local` (see the `herdr-macos27-zig-build` memory) — no env var
needed, `cargo install` finds `zig` on PATH.

### 9. Report what's new

Always finish with a themed digest of what came in. Don't paste `git log --oneline`
and call it a summary — group by what changed and why the user would care.

By step 9 the merge commit exists, so `git merge-base HEAD upstream/master` returns
`upstream/master` itself, not the base. Recover the real base from the merge commit's
two parents:

```bash
BASE=$(git merge-base HEAD^1 HEAD^2)          # HEAD^1 = local tip, HEAD^2 = upstream tip
git diff $BASE upstream/master -- CHANGELOG.md            # upstream's own framing
git log --format='%s' $BASE..upstream/master | sed 's/ (#[0-9]*)$//' | sort
git log --oneline $BASE..upstream/master | grep -n 'release: v'   # release boundary
```

**Two traps that produce a wrong digest:**

- **The changelog overstates novelty.** A `## [x.y.z]` section is written *at release
  time*, so diffing it surfaces the whole section — including features that arrived in
  an earlier sync and were already on the branch. Only commits in `$BASE..upstream/master`
  are actually new. Check entries against that range and say which ones the branch
  already had, rather than reciting the changelog as if all of it is new.
- **Split at the `release: vX.Y.Z` commit.** Its ancestors shipped in that release;
  commits after it are unreleased upstream work. Separate them — the user tracks a
  released version, so "what I was missing from the release" and "what's unreleased"
  are different answers. Note the version bump too (`cargo install` prints old → new).

Group the body by theme, not commit order: features, perf, correctness (usually the
densest), platform-specific, refactors, licensing/process. One line each, keeping
upstream's issue numbers.

**Then the part that matters most:** a short list of the upstream commits that land on
or beside the local features — cross-referenced from the conflicts just resolved and
the deleted-local-code audit. That tells the user where to hand-test. Explicitly flag
overlap a clean compile and green tests cannot verify, UI/visual changes especially,
instead of implying validation covered it.

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

- **Pinned toolchain**: upstream ships a `rust-toolchain.toml` (currently
  `channel = "1.96.1"`). After fetching, glance at it (`cat rust-toolchain.toml`);
  if the pin moved, `rustup` auto-installs the channel on the next cargo command.
  The local macOS-27 zig build setup is independent of the Rust channel and keeps
  working.
