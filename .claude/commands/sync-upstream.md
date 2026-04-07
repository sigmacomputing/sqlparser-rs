# Sync Upstream into Sigma Fork

Sync the sigmacomputing fork of sqlparser-rs with commits from the apache upstream remote.

## Instructions

### Step 1: Determine the sync target

The user may have provided a sync target as an argument: `$ARGUMENTS`

If a target was provided, interpret it as one of:
- `latest` or empty — sync to `upstream/main` HEAD
- `latest-tag` — sync to the most recent upstream tag
- A tag name like `v0.61.0` — sync to that specific tag
- A commit hash — sync to that specific commit

If no argument was provided, first fetch remotes and gather context, then ask the user:
```bash
git fetch origin
git fetch upstream --tags

# Gather info for the prompt:
git log -1 --format="%h %s (%ci)" upstream/main          # latest commit
git tag --list --sort=-version:refname | grep -E '^v?[0-9]' | head -1  # latest tag name
git log -1 --format="%h %s (%ci)" $(git tag --list --sort=-version:refname | grep -E '^v?[0-9]' | head -1)  # latest tag commit info
```

Use AskUserQuestion with the gathered info included in each option description:
1. **Latest commit on upstream/main** — show the commit hash, message, and date
2. **Latest tag** — show the tag name, commit hash, and date
3. **Specific tag** — prompt for tag name (list a few recent tags with dates as context)
4. **Specific commit** — prompt for commit hash

Once the target is known, resolve it:
```bash
# Latest commit:   TARGET=upstream/main
# Latest tag:      TARGET=$(git tag --list --sort=-version:refname | grep -E '^v?[0-9]' | head -1)
# Specific tag:    TARGET=<tag>
# Specific commit: TARGET=<hash>

git rev-parse $TARGET   # confirm it resolves
```

### Step 2: Create a new branch off the target

Name the branch to reflect what's being synced:
```bash
git checkout -b ayman/sync-upstream-$(date +%Y%m%d) $TARGET
```

### Step 3: Merge origin/main
```bash
git merge origin/main --no-ff -m "Merge sigma origin/main into upstream/main for sync"
```

This will likely produce conflicts. Do NOT abort — proceed to resolve them.

### Step 4: Resolve all merge conflicts

For each conflicted file:

1. **Understand sigma's intent**: Look at the original sigma commits and PRs to understand the purpose behind each change. Use `git log`, `git show`, and the GitHub PR history as needed.

2. **Understand upstream's changes**: Determine whether upstream's changes cover, supersede, or are orthogonal to sigma's changes.

3. **Default resolution**: Keep changes from BOTH sides unless one side clearly supersedes the other. Never discard either side without good reason.

4. Ask the user if uncertain about any conflict resolution.

After resolving all conflicts in all files:
```bash
git add <all-resolved-files>
git commit --no-edit   # completes the merge commit
```

All conflict resolutions must be in the single merge commit.

### Step 5: Fix compilation and test issues

```bash
cargo clippy --all-features --all-targets
```

Investigate and fix any errors or warnings. Commit fixes as a separate commit (do not amend the merge commit).

### Step 6: Run full verification

```bash
cargo fmt
cargo clippy --all-features --all-targets
cargo test --all-features --all-targets
```

All must pass before creating the PR. Fix any remaining issues and commit them.

### Step 7: Push and create PR

```bash
git push origin ayman/sync-upstream-$(date +%Y%m%d)
gh pr create \
  --repo sigmacomputing/sqlparser-rs \
  --base main \
  --head ayman/sync-upstream-$(date +%Y%m%d) \
  --title "[chore] Sync upstream apache/datafusion-sqlparser-rs into sigma fork" \
  --body "..."
```

The PR description must include:
- The upstream ref being synced to (commit hash, tag, or HEAD) and what it corresponds to
- Number of upstream commits brought in
- Summary of notable upstream changes (new features, dialects, fixes)
- List of all sigma features confirmed preserved
- Summary of each conflict resolved and how
- Checklist confirming clippy, fmt, and tests all pass
