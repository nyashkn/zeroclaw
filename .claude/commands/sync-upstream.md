# sync-upstream

Sync `main-track` with the latest `origin/main` upstream, preserving only the local-unique commits on top.

## When to use

Run this whenever the upstream `zeroclaw-labs/zeroclaw` has advanced and you want to pull in all upstream changes without losing local fixes.

## Steps

### 1. Identify local-unique commits

```bash
git fetch origin
git log origin/main..HEAD --oneline
```

Commits **without** an upstream PR number (e.g. no `(#NNNN)`) are the ones to re-apply after the merge. Save their hashes.

### 2. Save patches for local-unique commits

```bash
git show <hash> -- <file> > /tmp/patch-<name>.diff
```

### 3. Tag current state (safety net)

```bash
git stash          # stash any uncommitted changes
git tag backup/main-track-pre-merge
```

### 4. Merge upstream — take theirs for all conflicts

```bash
git merge origin/main --no-ff -m "merge: sync with upstream origin/main"

# Resolve every conflict by accepting upstream:
git diff --name-only --diff-filter=U | while read f; do
  git checkout --theirs "$f" 2>/dev/null && git add "$f" && continue
  git rm "$f" 2>/dev/null && continue
  echo "SKIP: $f"
done

# Handle rename/rename artifacts (images etc.)
git diff --name-only --diff-filter=U   # should be 0

git commit
```

### 5. Re-apply local-unique commits

```bash
git cherry-pick <hash1> <hash2> ...
# If a cherry-pick is empty (already in upstream): git cherry-pick --skip
```

### 6. Fix post-merge build errors

```bash
cargo check 2>&1 | grep "^error\["
```

Common patterns after an upstream sync:
- Renamed methods → compiler suggests the new name
- Missing struct fields → add with a sensible default
- Non-exhaustive match → add the new variant arm
- Removed crates → replace with `std` equivalent

### 7. Validate & push

```bash
cargo check
cargo test
git push fork main-track
```

### 8. Restore stash

```bash
git stash pop
```

---

## Key principle

> **Always take upstream for all conflicts.** Your local branch has very few unique commits — it is cheaper to re-apply them on top of a clean upstream merge than to fight hundreds of conflict markers.
>
> Never take HEAD for `mod.rs` / `lib.rs` / `main.rs` — those are module-wiring files that upstream owns.
