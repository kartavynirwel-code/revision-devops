# Day 1 — Git (Advanced) + Bash Scripting Revision Notes

## 1. Git Advanced Commands

### `git rebase -i` (Interactive Rebase)
- **What it does**: Rewrites commit history by letting you pick, squash, reword, edit, or drop commits.
- **Command**: `git rebase -i HEAD~n` (last n commits) or `git rebase -i <commit-hash>`
- **Options in the editor**:
  - `pick` — keep commit as is
  - `reword` — keep changes, edit commit message
  - `squash` — merge into previous commit, combine messages
  - `fixup` — merge into previous commit, discard this message
  - `drop` — remove commit entirely
  - `edit` — pause rebase to amend that commit
- **Why it matters (interview angle)**: Used to clean up messy commit history before a PR merge — squash "WIP" commits into one meaningful commit. Shows discipline in commit hygiene.
- **Danger**: Never rebase commits that have already been pushed and pulled by others — it rewrites SHA hashes, causing history divergence for teammates.

### `git cherry-pick`
- **What it does**: Applies a specific commit from one branch onto another, without merging the whole branch.
- **Command**: `git cherry-pick <commit-hash>`
- **Use case**: Hotfix on `main` needs to also go into `release/v2` branch — cherry-pick that one commit instead of merging everything.
- **Conflict handling**: If conflict occurs, resolve manually, then `git cherry-pick --continue` (or `--abort` to cancel).

### `git reflog`
- **What it does**: Shows a log of *all* HEAD movements — commits, resets, rebases, checkouts — even ones not visible in `git log` (like commits from a branch you deleted or reset away from).
- **Command**: `git reflog`
- **Why it matters**: It's your safety net. If you did a bad `git reset --hard` and lost commits, `git reflog` shows the old HEAD position, and you can recover with `git reset --hard <reflog-entry>` or `git checkout <reflog-entry>`.
- **Interview point**: Reflog is local-only (not pushed to remote), and entries expire after ~90 days by default.

### `git stash`
- **What it does**: Temporarily shelves uncommitted changes (staged + unstaged) so you can switch branches cleanly, then reapply later.
- **Commands**:
  - `git stash` or `git stash push -m "message"` — save changes
  - `git stash list` — see all stashes
  - `git stash pop` — apply latest stash and remove it from stash list
  - `git stash apply` — apply but keep it in the stash list
  - `git stash drop` — delete a specific stash
  - `git stash -u` — also stash untracked files
- **Use case**: Mid-feature work, urgent bug comes in on another branch — stash, switch, fix, switch back, pop.

### `git bisect`
- **What it does**: Binary search through commit history to find which commit introduced a bug.
- **Commands**:
  - `git bisect start`
  - `git bisect bad` (mark current commit as broken)
  - `git bisect good <commit-hash>` (mark a known-good commit)
  - Git checks out a middle commit — you test it, mark `git bisect good` or `git bisect bad`
  - Repeats until the exact bad commit is found
  - `git bisect reset` — end the session
- **Why it matters**: Much faster than manually checking out commits one by one in a large history. Can be automated with `git bisect run <script>` if you have a test script that exits non-zero on failure.

---

## 2. Bash Scripting

### Loops
```bash
# for loop
for i in {1..5}; do
  echo "Iteration $i"
done

# while loop
count=0
while [ $count -lt 5 ]; do
  echo "Count: $count"
  ((count++))
done

# looping over command output
for pod in $(kubectl get pods -o name); do
  echo "$pod"
done
```

### String Manipulation
```bash
str="DevOps-Engineer"
echo "${str,,}"        # lowercase → devops-engineer
echo "${str^^}"        # uppercase → DEVOPS-ENGINEER
echo "${str/DevOps/Java}"   # replace first match → Java-Engineer
echo "${str//e/E}"     # replace all matches
echo "${str#*-}"       # remove shortest match from front → Engineer
echo "${str%-*}"       # remove shortest match from back → DevOps
echo "${#str}"         # length of string
```

### Exit Codes
- Every command returns an exit code: `0` = success, non-zero = failure.
- Check with `echo $?` immediately after a command.
- Use in conditionals:
```bash
if command; then
  echo "Success"
else
  echo "Failed with code $?"
fi
```
- Custom exit codes in your own scripts: `exit 1`, `exit 2`, etc. — useful for CI/CD pipelines to distinguish failure types.

### `trap`
- **What it does**: Catches signals (like `EXIT`, `SIGINT`, `SIGTERM`) and runs cleanup code.
- **Example**:
```bash
cleanup() {
  echo "Cleaning up temp files..."
  rm -f /tmp/mytempfile
}
trap cleanup EXIT

# Also common:
trap 'echo "Interrupted"; exit 1' SIGINT
```
- **Why it matters (DevOps angle)**: Used in CI scripts to always clean up resources (temp files, locks, containers) even if the script fails midway or is interrupted.

### Debugging a Script
- `bash -x script.sh` or add `set -x` inside the script — prints each command before executing it (trace mode).
- `set -e` — exit immediately if any command fails (fail-fast, very common in CI scripts).
- `set -u` — treat unset variables as an error.
- `set -o pipefail` — makes a pipeline fail if *any* command in it fails, not just the last one.
- Common combo at the top of production bash scripts:
```bash
#!/bin/bash
set -euo pipefail
```
- **Interview point**: Explain `set -euo pipefail` as the "strict mode" for bash — prevents silent failures in automation scripts.

---

## Quick Self-Test (do this without looking)
1. How do you recover a commit after an accidental `git reset --hard`?
2. Difference between `git stash apply` and `git stash pop`?
3. What does `set -o pipefail` actually fix that `set -e` alone doesn't?
4. Write a one-liner to loop through all `.log` files in a directory and print their line count.
