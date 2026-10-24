# Remote-state check (steps 3 and 4)

Loaded from `git-merge/SKILL.md`. Read this file at step 3. The step-4 choice question and the Hard rules stay in `SKILL.md` and remain in force throughout.

## Step 3: targeted fetch

First handle a remote-tracking source on a remote other than `origin` (step-1 classification, for example `template/main`): this skill never fetches that remote. Run `git rev-parse --verify --quiet <source-ref>`; when it prints nothing, STOP and tell the user to run `git fetch <remote>` themselves, then re-run this skill. When it resolves, note in the step-9 plan that `<source>` is as of the last `git fetch <remote>`, and that running `git fetch <remote>` first picks up newer commits.

Then use the step-1 `git remote` output. When it does not list `origin` (a local-only repo, common under `trunk-solo`), skip the rest of step 3 and all of step 4, and note "no origin remote; merging local refs as-is" in the step-9 plan.

Each side of the merge is a *local* ref (`main`, `feature/foo`) or a *remote-tracking* ref (step-1 classification). Target is always local (step 2 aligned HEAD with it); source may be either form (`origin/<branch>` in a suggested divergence merge).

Collect the local-side refs only: `<target>` always, plus `<source>` when it is a local source. If the collected list is empty (both sides already remote-tracking, unusual but possible), skip the fetch entirely; those refs ARE the remote and need no refresh. Otherwise fetch each collected ref with its own command, one call per ref:
```
git fetch --no-tags origin +refs/heads/<ref>:refs/remotes/origin/<ref>
```
The explicit refspec writes `origin/<ref>` even in a `--single-branch` or shallow clone. Its `+` force-updates only that remote-tracking ref; `--no-tags` stops the tag auto-follow an explicit refspec triggers. Per-ref calls are deliberate: a ref with no counterpart on `origin` exits 128 with `couldn't find remote ref`, and a combined call would fail as a whole. Route each ref's outcome:
* Exit 0: tracking ref refreshed; the ref is eligible for the step-4 check.
* Exit 128 with `couldn't find remote ref`: record that ref as "no remote tracking" for step 4 and proceed.
* Any other non-zero exit (network, auth, proxy): surface stderr verbatim and STOP.

This is read-only against the working tree.

## Step 4: divergence classification

Run the check only for local-side refs whose step-3 fetch succeeded (exit 0). A ref recorded as "no remote tracking" is skipped even when a stale `origin/<X>` lingers locally from before the remote branch was deleted; comparing against that stale ref would misreport divergence. Skip any side that is a remote-tracking ref: it IS the remote (for a remote other than `origin`, as of its last fetch). In the common divergence merge that `git-pull` or `git-push` suggests (source=`origin/<branch>`, target=`<branch>`), only the target is checked.

For each eligible ref, run these independent reads: `git rev-list --left-right --count refs/heads/<branch>...refs/remotes/origin/<branch>`. Then classify:
* **0 / 0**: local matches remote; no action.
* **N / 0** (local ahead): unpublished commits; informational only.
* **0 / N** (local behind): STALE. Merging here lands the merge commit on a stale tip and forces a follow-up integration before the next push.
* **M / N** (diverged): merging would miss remote commits on that branch.

Findings to render when any eligible ref is stale or diverged: the ahead / behind counts plus `git log refs/heads/<branch>..refs/remotes/origin/<branch> --oneline` capped at 10 lines per ref, so the user sees what would be missed.
