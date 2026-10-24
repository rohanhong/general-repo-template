# Repository provenance: a clone of the template

Loaded from `repo-init/SKILL.md`. Read this file at step 2 when the repository is a clone of the template, before asking. The Hard rules in `SKILL.md` remain in force throughout.

## Evidence

Render the evidence before the question: the origin URL, `git log --oneline -5`, the total commit count from `git rev-list --count HEAD`, the local-only commits from `git log --branches --tags HEAD --not --remotes --oneline` (reachable only from a local branch, tag, or detached HEAD), `git stash list`, `git worktree list --porcelain` (every `worktree` entry after the first is a linked worktree), the local branch and tag names origin lacks, and the submodule repositories stored inside `.git` with their local-only commits:

```
sh <<'EOF'
r=$(GIT_TERMINAL_PROMPT=0 git ls-remote --refs origin 'refs/heads/*' 'refs/tags/*') || { echo "origin unreachable: count every local branch and tag name as local-only"; exit 0; }
{ printf '%s\n' "$r" | sed 's/^/R /'; git for-each-ref --format='L %(refname)' refs/heads refs/tags; } | awk '$1=="R"{s[$3]=1;next} !($2 in s){print "local-only name: " $2}'
m="$(git rev-parse --git-common-dir)/modules"
if [ -d "$m" ]; then find "$m" -type d -name objects | while read -r o; do d=${o%/objects}; [ -f "$d/HEAD" ] || continue; echo "submodule repository inside .git: ${d#"$m"/}"; git --git-dir="$d" log --oneline --branches --tags HEAD --not --remotes | sed 's/^/  local-only: /'; done; fi
EOF
```

## Options

* **fresh start**: `rm -rf .git`, then `git init -b main` (older git: `git init && git symbolic-ref HEAD refs/heads/main`). Deletes the inherited template history and remote; every working file stays untouched. This is the one deliberately destructive action in this skill, so the question text itself MUST restate exactly what is deleted, INCLUDING every local-only commit on any branch, tag, or detached HEAD, every local-only branch or tag name, every stash entry, the git data of every linked worktree (those directories are left orphaned, no longer usable as worktrees), and every submodule repository stored inside `.git` with its local-only commits. Afterwards re-run the step-1 probe (see its skip rule in `SKILL.md`): the new `.git` has no `origin`, so 6a offers the remote wiring. When the local-only log, the local-only names, or the stash list is non-empty, or a linked worktree or submodule repository exists, do not proceed until the user confirms those are disposable.
* **keep history**: the user wants to pull template updates later, or their own commits must survive. Keep `.git` as-is, and offer a gated `git remote rename origin template`, freeing the `origin` name for the project's own remote. After the rename, treat `origin` as absent for the rest of this run, so 6a offers the remote wiring. For later template updates, the user runs `git fetch template`, then suggest `git-merge` with source `template/main`.
* **template itself**: this repository IS the template, or a fork the user maintains as a template. Proceed as keep history WITHOUT the rename offer (`origin` already is the project's own remote), and step 7 treats the repo as the template itself.
* **abort**: STOP.
