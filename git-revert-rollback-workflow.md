# Git Rollback Workflow — Reverting a Merged PR

This summarizes the rollback process practiced in the `git-lab-practice` repo: undoing merged pull requests on `main` using `git revert`, without rewriting shared history.

## Why `git revert` instead of `git reset`

- **`git revert`** creates a *new* commit that undoes the changes from an old commit. History stays intact — safe for branches others have pulled from (like `main`).
- **`git reset`** deletes commits from history. This causes problems if others have already pulled that code, since their local history no longer matches. Reserved for local, unpushed mistakes only.

**Rule of thumb:** local/unpushed → `reset` is fine. Pushed/merged/shared → always `revert`.

## Step-by-Step Workflow

### 1. Sync with the remote
```cmd
git checkout main
git pull
```
- `checkout main` — switches your working directory to the `main` branch.
- `pull` — fetches and merges the latest commits from GitHub into your local `main`, so you're working from the current state before making any changes.

### 2. Find the merge commit to undo
```cmd
git log --oneline --merges
```
- `log` — shows commit history.
- `--oneline` — condenses each commit to one line (hash + message) for readability.
- `--merges` — filters the log to show **only** merge commits (i.e., commits created when a PR was merged), making it easy to spot the one you want to undo.

Optional double-check on a specific commit:
```cmd
git show <commit-sha> --stat
```
- `show` — displays details of a specific commit.
- `--stat` — instead of showing full file diffs, just lists which files changed and how many lines, giving a quick summary.

### 3. Identify the correct commit — only revert merges into the branch you're on
A merge commit only qualifies for this workflow if it represents a PR merged **into `main`**. Merge commits that occurred on a feature branch (e.g., merging `main` *into* a feature branch to resolve a conflict) are not valid revert targets from `main` — reverting them undoes the wrong side of history and often causes confusing conflicts.

### 4. Revert the commit(s)
For a single merge:
```cmd
git revert -m 1 <merge-commit-sha>
```
- `revert` — creates a new commit that reverses the changes of the target commit.
- `-m 1` — required specifically for merge commits, which have two parents. This tells Git: "treat parent #1 (the branch you're on, e.g. `main`) as the baseline, and undo the changes that came in from parent #2 (the merged branch)."

**To roll back to the state as of an *older* merge** (undoing that merge plus everything after it), revert in **reverse chronological order** — most recent commit first:
```cmd
git revert -m 1 <most-recent-merge-sha>
git revert -m 1 <older-merge-sha>
```
Reverting newest-to-oldest avoids conflicts between overlapping changes.

Each revert opens your default text editor with a pre-filled commit message (e.g., `Revert "Merge pull request #4..."`). Save and close it:
- **Notepad:** `Ctrl+S`, then close the window.
- **Vim:** press `Esc`, type `:wq`, press Enter.

### 5. Handle conflicts (if they occur)
```cmd
# open the flagged file(s), fix the <<<<<<< ======= >>>>>>> markers
git add <filename>
git revert --continue
```
- `add` — stages the resolved file.
- `revert --continue` — tells Git the conflict is resolved and to finish creating the revert commit.

To back out of a revert entirely and return to the pre-revert state:
```cmd
git revert --abort
```

### 6. Push the revert commit(s)
```cmd
git push origin main
```
- Sends your new revert commit(s) to the remote `main` branch on GitHub.
- If branch protection rules require PRs (no direct pushes to `main`), push to a new branch instead and open a PR:
```cmd
git checkout -b revert-fix
git push -u origin revert-fix
```
Then merge that PR through GitHub as usual.

### 7. Verify the result
```cmd
git log --oneline
```
Confirms the new revert commit(s) sit on top of the history, and the affected file(s) should now reflect the pre-merge (or pre-mistake) content.

## Common Pitfall Encountered

Reverting the **wrong merge commit** (e.g., the very first merge that added a file) can cause a conflict like:
```
CONFLICT (modify/delete): notes.txt deleted in parent of <sha> and modified in HEAD.
```
This happens because Git is trying to undo the file's *creation*, but the file has since been modified by later commits. Fix: `git revert --abort`, then re-identify the correct, more recent merge commit to target.
