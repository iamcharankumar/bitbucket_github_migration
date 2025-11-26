# How I Moved a 2-Million-Line Bitbucket Universe to GitHub Without Losing a Single Atom of History?

## Disclaimer

```textmate
This article describes a generic, publicly documented Git migration workflow.  
It does not reference or disclose any proprietary code, filenames, repositories, 
internal tooling, or confidential information of any employer or organization.

All examples are illustrative and recreated for educational purposes only.  
Readers should verify all steps in a safe, non-production environment before use.
```

Here’s How This Monstrous Migration Actually Happened

Migrating a tiny repo is easy. Just use the URL: http://github.com/new/import

Migrating a mid-sized repo is doable.

But migrating a massive, six-digit-scale enterprise repo packed with ~50K commits, over 2.5K branches, 150+ authors,
thousands of PR histories, timestamps ranging across years, and nearly 2 million lines of code—without losing a
single branch, author, commit, or line-change fingerprint?

That’s not a migration.

That’s **a full-scale Git rescue mission**.

This blog breaks down exactly how that stunt was executed end-to-end:

- from solving macOS case-sensitivity issues with `hdiutil`,
- to cloning a repo that refused to clone,
- to surgically removing large files with BFG and solving LFS issues,
- to force-pushing a perfectly clean, fully traceable Git history into GitHub…

All without breaking a single reference.

This post tells the exact, copy-paste-ready sequence that fixed everything.
If you follow these steps exactly, you’ll get the same result.

Executive Summary (one-sentence TL;DR)

> Create a case-sensitive workspace → mirror-clone the repo → surgically remove historic large artifacts using BFG →
> garbage-collect and compact → reattach remote → mirror-push to GitHub → Done!

# Key Migration Metrics Table

| Sl. No | Metric                                 | Command                                                          | Why It Matters                             |
|--------|----------------------------------------|------------------------------------------------------------------|--------------------------------------------|
| 1      | Total commits preserved                | `git rev-list --all --count`                                     | Ensures full historical continuity         |
| 2      | Total Branches (local + remote)        | `git for-each-ref --format='%(refname)' refs/remotes  \| wc -l`  | Zero loss of parallel development          |
| 3      | Total authors preserved                | `git log --format='%aN' \| sort -u \| wc -l`                     | Contributor integrity intact               |
| 4      | Number of PRs & Line Changes Preserved | `git diff --shortstat $(git rev-list --max-parents=0 HEAD) HEAD` | PR history + diff fidelity                 |
| 5      | Total refs (everything)                | `git show-ref \| wc -l`                                          | Guarantees complete reference preservation |
| 6      | Oldest commit timestamp                | `git log --reverse --format="%ad" \| head -1`                    | Verifies root history intact               |
| 7      | Latest commit timestamp                | `git log -1 --format="%ad"`                                      | Confirms continuity post-migration         |

# Atomic tasks — the exact sequence you must follow

> Important: Replace placeholders shown in angle brackets (<>) with your actual values.
> Do everything from a machine
> with enough disk and RAM for the repo.

<hr>

## 1) Create a case-sensitive workspace (macOS only) — solve branch case-collisions

**Why**: macOS default filesystems are case-insensitive.
If your Bitbucket repo has branch names that differ only by case (e.g. Feature vs feature),
a case-insensitive filesystem will break tools that expect case sensitivity.
Working inside a case-sensitive disk image avoids those collisions.

```shell
# create a 40 GB case-sensitive disk image and mount it
hdiutil create -size 40g -type SPARSEBUNDLE -fs "Case-sensitive APFS" -volname GitCS ~/GitCS.sparsebundle
hdiutil attach ~/GitCS.sparsebundle
# now work inside /Volumes/GitCS
cd /Volumes/GitCS
```

When finished, detach with: _(Do it at the end, not immediately after creating the disk image.)_

```shell
hdiutil detach /Volumes/GitCS
```

# 2) Mirror-clone the entire Bitbucket repository (bare clone for surgery)

**Why**: `--mirror` clones all refs (branches, tags, remote refs) — exactly what you need to rewrite history globally.

```shell
# inside /Volumes/GitCS (the case-sensitive volume)
git clone --mirror <OLD_BITBUCKET_REPO_URL> myrepo.git
cd myrepo.git
```

`myrepo.git` is a bare repo; it has no working tree and is suitable for repository-wide surgery.

# 3) Identify the patterns you need to remove (audit first)

**Why**: Don’t shotgun-delete.
Identify the offending patterns (e.g., a set of test data dumps or giant artifacts) to avoid
collateral damage.

Example ways to inspect (run in the bare mirror repo parent):

```shell
# show top largest blobs in repo history (helpful for discovery)
git rev-list --objects --all \
  | git cat-file --batch-check='%(objecttype) %(objectsize) %(rest)' \
  | grep blob | sort -n -k2 | tail -20
```

You can also search commit content names:

```shell
# show top largest blobs in repo history (helpful for discovery)
git rev-list --all | while read commit; do git ls-tree -r --name-only $commit | grep "<pattern-or-identifier>" || true; done
```

Record the pattern(s) you will remove — call it `<PATTERN>`.

# 4) Install BFG Repo-Cleaner

**Why**: BFG is fast and battle-tested for deleting unwanted files from history.

```shell
brew install bfg
bfg --version
```

If `bfg` complains about Java, install/point to a compatible Java runtime (Homebrew wrapper handles most cases).

# 5) Run BFG to delete the target patterns from all history

**Why**: This rewrites every commit that contains the offending objects and removes the objects from history.

> Important: BFG matches by filename pattern (not full path). Use a pattern that matches the offending artifact names.
> Replace <PATTERN> with that glob (e.g. `BIG_DATA_*.bin` or `OLD_DUMP_*.csv`).

```shell
# Run from the parent of the bare repo:
cd /Volumes/GitCS
bfg --delete-files '<PATTERN>' myrepo.git
```

**Expected BFG output:** lists of deleted blobs and the commits touched.
If BFG errors due to extreme blob sizes, rerun the
analysis and consider refining the pattern; in our run BFG was the tool that completed the cleanup successfully (after
we fixed case-sensitivity).

# 6) Expire reflogs and run Git GC to permanently remove blobs

**Why:** BFG rewrote history but Git still keeps unreachable objects until cleanup.

```shell
# Run from the parent of the bare repo:
cd myrepo.git
git reflog expire --expire=now --all
git gc --prune=now --aggressive
```

Confirm repo shrank and large blobs are gone.

# 7) Verify cleanliness before pushing

**Why:** Double-check there are no >100MB blobs left (GitHub’s hard limit), and confirm commit counts and refs.

Commands to run in the bare repo (or a working clone pointing to it):

```shell
# total commits
git rev-list --all --count

# list any blobs > 100MB remaining (should be none)
git rev-list --objects --all \
  | git cat-file --batch-check='%(objecttype) %(objectsize) %(rest)' \
  | awk '$2 > 100000000 {print}'
```

If any large blobs remain, repeat Step 3–6 with refined patterns until empty.

# 8) Add the GitHub remote and push the cleaned repo

**Why:** A bare mirror may have lost or need a remote; pushing --mirror reproduces all cleaned refs at the destination.

```shell
# inside the bare repo
git remote add origin <NEW_GITHUB_REPO_URL>
git remote -v

# final push of the entire cleaned repo
git push origin --mirror
```

To use an SSH key with an organization that uses single sign-on (SSO),
please follow the steps
given [here](https://docs.github.com/en/enterprise-cloud@latest/authentication/authenticating-with-single-sign-on/authorizing-an-ssh-key-for-use-with-single-sign-on#authorizing-an-ssh-key).

# 9) Optional — If some refs are protected and reject force pushes

**Why:** Bitbucket or GitHub pre-receive hooks may reject force-pushes for protected branches.

You can:

- Push only cleaned branches you control, e.g. git push origin refs/heads/master --force,
- Or Ask org admins to temporarily relax protection,
- Or Upload a git bundle and import it via GitHub import.

Example: push just `master`:

```shell
git push origin refs/heads/master:refs/heads/master --force
```

# 10) Post-migration validation (what I ran)

Commands I ran to validate everything you see at the top:

```shell
# total commits preserved:
git rev-list --all --count

# total branches (local+remote):
git for-each-ref --format='%(refname)' refs/remotes | wc -l

# total authors:
git log --format='%aN' | sort -u | wc -l

# PR diffs / lines preserved:
git diff --shortstat $(git rev-list --max-parents=0 HEAD) HEAD

# total refs:
git show-ref | wc -l

# oldest commit:
git log --reverse --format="%ad" | head -1

# newest commit:
git log -1 --format="%ad"
```

These are the exact commands whose outputs were assembled into the metrics table above.

## Final validation: cloc for lines of code

To install cloc on your machine, please read this article
[here](https://www.geeksforgeeks.org/linux-unix/cloc-count-number-of-lines-of-code-in-file/).

After cloning the cleaned GitHub repo to a working tree, I ran:

```shell
git clone <NEW_GITHUB_REPO_URL>
cd <repo>
cloc .
```

**You'll see an output similar to the below image based on your repo's lines of code.
So do it before and after migration for better validation.**

# Post-mortem checklist

- All branches preserved and mapped to original owners (no orphaned branches) ✅
- Commit history count preserved (full history) ✅
- PR diffs and per-PR line counts preserved (no loss of metadata) ✅
- Large historic blobs removed and repo compacted ✅
- Developers instructed to reclone (after a history rewrite, reclone is simpler than fixing remotes) ✅
- Migration logs saved and stored for audit ✅

# Troubleshooting notes (quick hits)

- If bfg fails due to Java errors, ensure a compatible Java runtime or use the Homebrew wrapper.
- If git push --mirror fails with pre-receive hook declined,
  check branch protection / pre-receive rules and push only the
  cleaned, non-protected branches or ask admins to temporarily allow the push.
- If cloning via SSH for an enterprise org returns an SAML SSO error, authorize your SSH key or use an HTTPS PAT that's
  SAML-authorized.
- If git filter-repo is preferred, it can work — but in our run BFG completed the cleanup reliably after the
  case-sensitive workspace fix.

Closing — the moment that mattered

A high-stakes, surgical migration that preserved every last commit
and branch, mapped ownership cleanly from Bitbucket to GitHub, and left the organization with a compact, clean repo and
an auditable migration trail.
If you’re about to attempt the same: prepare the case-sensitive environment first, pick
your removal patterns carefully, and use BFG for the heavy lifting.
Then sit back while `git push --mirror` does the happy final dance.

**Finally, the test automation repo migration was completed nice and clean, moving from Bitbucket to GitHub!**

# EXTRAS

## Why BFG, not git filter-repo?

I tried all of these earlier, remember?

- git filter-repo --path-glob ...
- git filter-repo --path ... --invert-paths
- git filter-branch (didn’t use because it's too slow)

Everything _failed_ because:

1. The repo was huge (enormous thousands of commits; `GitHub history size ≠ local branch commit count`).
2. The working directory and branch names were clashing due to macOS case-insensitive FS.
3. `git filter-repo` kept choking on large blobs.
4. BFG also initially crashed due to Java version mismatch
5. The `hdiutil` case-sensitive FS + BFG wrapper = **perfect stable combo**

So yes, the exact savior is BFG here.

# 🔥 BFG (run through Java 8 wrapper)

- Not git filter-repo.
- Not git bash magic.
- Just BFG cleaning all the history cleanly after fixing the FS issue.

# 💡 Why BFG works where filter-repo fails?

BFG internally scans history in a more optimized way by:

- Skipping unnecessary rewrite of commits
- Deleting entire blobs quickly
- Being comfortable with massive repos

`git filter-repo` is more powerful but way more picky and resource-hungry. My repo was simply too large.

BFG handled it cleanly once run with proper Java.