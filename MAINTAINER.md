# KissClawJJ Maintainer Guide

KissClawJJ uses **jj** (Jujutsu) in non-colocated mode as its primary VCS
workflow. Git is the backing store; jj manages the working copy and commit graph.

## Prerequisites

- jj 0.28+ installed (`jj version`)
- Git credentials configured for `github.com` (HTTPS via `gh auth`)
- The workspace at `~/git/MachineWisdomAI/kissclawjj` is a non-colocated jj
  workspace (`.jj/` exists, `.git/` does NOT)

## Key Commands

### Cherry-picking upstream commits

```bash
# Ensure upstream remote is configured
jj git remote list  # should show 'upstream' pointing at openclaw-org/openclaw

# Fetch latest upstream
jj git fetch --remote upstream

# Cherry-pick a specific upstream commit onto the current working copy
jj duplicate <upstream-sha> -d @

# If conflicts appear:
jj status           # shows conflicted files
jj resolve          # interactive conflict resolution
# Or edit files directly and jj squash

# Set commit message (preserve cherry-pick provenance)
jj describe -m "fix: description here

(cherry picked from commit <upstream-sha>)"

# Advance working copy for the next cherry-pick
jj new
```

### Managing bookmarks (branches)

```bash
# Create a new bookmark
jj bookmark set <name> -r <revision>

# Push a NEW bookmark (first push requires --allow-new)
jj git push --bookmark <name> --allow-new

# Push an existing bookmark (subsequent pushes)
jj git push --bookmark <name>

# Move a bookmark to current revision
jj bookmark set <name> -r @
```

### Release workflow

```bash
# 1. Ensure governance-patch is rebased onto main
jj rebase -b governance-patch -d main

# 2. Create ga/1.0 from governance-patch
jj bookmark set ga/1.0 -r governance-patch
jj git push --bookmark ga/1.0 --allow-new

# 3. Start release branch work
jj new ga/1.0

# 4. Cherry-pick fixes (see above)

# 5. After all cherry-picks, set release bookmark
jj bookmark set release/v1.0.1-rc.1 -r @
jj git push --bookmark release/v1.0.1-rc.1 --allow-new
```

### Pre-push validation

jj has no native pre-commit hook in non-colocated mode. Run validation manually
before pushing:

```bash
# Run kc-check-imports against the accumulated changes
kc-check-imports --baseline v2026.4.20 --candidates governance-patch..@ --repo .

# Also run final-tree validation
kc-check-imports --baseline v2026.4.20 --final-tree governance-patch..@ --repo .
```

### Force-push governance-patch

```bash
# After rebasing governance-patch onto a new main baseline
jj bookmark set governance-patch -r <new-tip>
jj git push --bookmark governance-patch
# jj handles force-push automatically when the bookmark has moved
```

## Important Notes

- **Never export unresolved jj conflicts to Git.** Always resolve before pushing.
- The CI workflow on `release/*` PRs is the authoritative validation gate.
- Local `kc-check-imports` runs are fast feedback, not a substitute for CI.
- When in doubt, use `jj log` to visualize the commit graph before pushing.
