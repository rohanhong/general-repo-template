# Main protection: the protect-main ruleset

Loaded from `repo-init/SKILL.md`. Read this file at step 6h, before asking. The Hard rules in `SKILL.md` remain in force throughout.

## What the file is

`.github/rulesets/protect-main.json` is a GitHub branch ruleset in the format GitHub's ruleset import and REST API accept. It targets `refs/heads/main` and holds two rules, `deletion` and `non_fast_forward`, with no bypass actors, so nobody, administrators included, can delete or force-push `main`. That is the server-side form of `.gitmessage`'s rule that forbids rewriting `main` under every workflow variant, so the file fits `git-flow`, `github-flow`, and `trunk-solo` alike.

GitHub does not read the file, and a repository created from the template does not inherit rulesets. The ruleset takes effect only once the user imports it.

## Step 6h procedure

Ask **protect** / **remove** / **keep for later**, with two narrower cases: when `origin` points to a forge other than GitHub, the file is GitHub-only, so offer only **remove** and **keep for later**; after a step-2 **template itself** answer, offer only **protect** and **keep for later**, since there the file is part of the template.

* **protect**: keep the file. When the variant from step 4 (or the recorded `Workflow:` line) is `github-flow` or `git-flow`, step 2 did not resolve to **template itself**, and the file's `rules` array has no `pull_request` entry yet, ask **add PR rule** / **no PR rule** for the pull request rule below (one required approval, as `.gitmessage` requires). On **add PR rule**, render the whole resulting file, then ask **write** / **abort**; on **write**, write the file and run the validity check below. Then render the import steps below for the user to run.
* **remove**: render the command below together with the diff that drops the `.github/rulesets/protect-main.json` bullet from the Conventions section of `AGENTS.md`, then ask **remove** / **abort**; on **remove**, run the command and apply that diff as an in-place edit:
  ```
  rm .github/rulesets/protect-main.json && { rmdir .github/rulesets 2>/dev/null || true; }
  ```
* **keep for later**: leave the file untouched; it has no effect until imported.

## Pull request rule (github-flow and git-flow)

Under `github-flow` and `git-flow`, `.gitmessage` also forbids direct pushes to `main`: updates land only through a pull request. The shipped ruleset does not enforce that. On **add PR rule** at step 6h, and only when `rules` has no `pull_request` entry yet (GitHub rejects a ruleset that repeats a rule type), add this object as a new element of the `rules` array:

```json
    {
      "type": "pull_request",
      "parameters": {
        "required_approving_review_count": 1,
        "dismiss_stale_reviews_on_push": true,
        "require_code_owner_review": false,
        "require_last_push_approval": false,
        "required_review_thread_resolution": false
      }
    }
```

The five parameters are the ones GitHub requires for this rule. `allowed_merge_methods` is left out, so the repository's own merge settings decide between merge commits and squash merges, both of which `.gitmessage` allows. One approval is the minimum `.gitmessage` sets for a pull request into a protected branch. GitHub does not let authors approve their own pull request, so a single maintainer is locked out by this rule; per `.gitmessage`, such a repository belongs on `trunk-solo`, where step 6h never offers it.

Validity check, run after the write: `python3 -m json.tool .github/rulesets/protect-main.json >/dev/null` or `jq empty .github/rulesets/protect-main.json`, whichever is installed. When neither is, re-read the file and confirm the new object sits inside `rules`, with a comma between elements and none after the last. On a failure, write back the content read before the edit and report; GitHub would otherwise reject the import.

Once this rule is active, pushing a local merge to `main` is rejected; `git-push` reports the rejection, and the change goes through a pull request instead.

Rules deliberately not added:

* `required_status_checks`: it names CI jobs, and the template has none. Add it on GitHub once CI exists, then copy the export back (see below).
* `required_linear_history`: it rejects merge commits, which `.gitmessage` and `git-merge` (`--no-ff`) produce.

Under `git-flow`, `develop` and `support/*` stay outside this ruleset; protecting them is a separate ruleset with its own name.

## Import steps (rendered for the user on protect)

Either path works; the user runs it, not the agent:

* On GitHub: Settings > Rules > Rulesets > New ruleset > Import a ruleset, then choose `.github/rulesets/protect-main.json`.
* With the GitHub CLI, from the repository toplevel (`gh api` fills in `{owner}` and `{repo}` from the current repository):
  ```
  gh api --method POST 'repos/{owner}/{repo}/rulesets' --input .github/rulesets/protect-main.json
  ```

Before rendering them, state these limits:

* Rulesets are not enforced on private repositories under GitHub Free; they need a public repository or a paid plan.
* GitHub documents that while force pushes are blocked, administrators cannot change or rename the default branch unless the ruleset lets them bypass it. With no bypass actors, edit or disable the ruleset first if GitHub refuses such a change.

## Keeping the file in sync

The file is the recorded source, as `.github/settings.yml` is for the About panel. A change made on GitHub is copied back: export the ruleset from its page on GitHub and replace the file with it, dropping the export-only fields `id`, `source`, and `source_type`. Rules added for `main` later, such as status checks, go into this same ruleset; the name `protect-main` stays accurate. Covering `develop` or `support/*` as well calls for a name that matches the wider scope.

## Why the shipped ruleset is minimal

* Pull request, review, and status check rules are variant-specific: `trunk-solo` allows direct commits on `main`, so the pull request rule is added only for `github-flow` and `git-flow`, at step 6h; status checks name CI jobs the template does not have.
* The target is the literal `refs/heads/main`, not `~DEFAULT_BRANCH`: `.gitmessage` fixes the trunk name as `main`, and a `git-flow` repository often sets `develop` as its default branch, which would leave `main` unprotected.
