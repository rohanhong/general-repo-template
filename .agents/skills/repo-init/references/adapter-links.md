# Adapter link states and repairs

Loaded from `repo-init/SKILL.md`. Read this file at step 1, which runs its probe script and reports the states it prints, and at step 2 before offering a register or repair gate. The Hard rules in `SKILL.md` remain in force throughout.

## Probe script

Run it as written, under `sh`, so an unmatched glob is safe in zsh too:

```
sh <<'EOF'
if git rev-parse --is-inside-work-tree >/dev/null 2>&1; then
  ex=$(git check-attr export-ignore -- .gitmessage)
  case "$ex" in *": set") echo "archive excludes: on";; *) echo "archive excludes: off";; esac
  git ls-files -s | awk '$1=="120000"{print $4}' | while read -r l; do
    t=$(git cat-file -p ":$l" | tr -d '\r\n')
    case "$t" in *.agents/skills) ;; *) continue;; esac
    [ "$(git cat-file -s ":$l")" -eq "${#t}" ] || echo "malformed: $l"
    { [ -L "$l" ] && [ -e "$l" ]; } || echo "broken: $l"
    case "$ex" in *": set")
      d=${l%%/*}
      case "$(git check-attr export-ignore -- "$d/")" in *": set") ;; *) echo "not export-ignored: $d/";; esac;;
    esac
  done
else
  echo "archive excludes: n/a (not a git repository)"
fi
for f in .[!.]*/skills; do
  [ -f "$f" ] && [ ! -L "$f" ] || continue
  case "$(tr -d '\r\n' < "$f")" in *.agents/skills) ;; *) continue;; esac
  [ "$(git ls-files -s -- "$f" 2>/dev/null | cut -d' ' -f1)" = 120000 ] && continue
  if [ "$(grep -c '' "$f")" -eq 1 ]; then echo "unregistered: $f"; else echo "malformed: $f"; fi
done
EOF
```

## What the probe identifies

Adapter links are identified by rule, never by path: a tracked symlink, or a `.<tool>/skills` plain file, whose target or content ends in `.agents/skills` once line breaks are stripped (AGENTS.md, Layout). Other symlinks in the project are ignored. `.gitmessage` is the sentinel for whether the `.gitattributes` export-ignore lines are enabled; the template ships them commented out (step 6g).

The probe runs under `sh` because zsh, the default interactive shell on macOS, aborts a command whose glob matches nothing (its NOMATCH option), while a POSIX `sh` (bash in POSIX mode, dash) leaves the unmatched `.[!.]*/skills` pattern literal, and the `[ -f "$f" ]` test then skips it.

## Probe output states

* `broken: <path>`: tracked as a link, but git wrote it as a plain file because symlinks were off at checkout, or its target is missing.
* `unregistered: <path>`: a link that arrived as a plain file through a copy or a fresh `git init` (the template's files were copied on a system without symlink support, or step 2's fresh start dropped the index that carried the link mode). A later `git add` would commit it as a regular file and lose the link for good; step 2 ends by registering it.
* `malformed: <path>`: the link target is not exactly one line ending in `.agents/skills`. For a tracked link, its stored target carries a line break (a trailing newline or CRLF from an editor or `echo`), so every symlink-aware checkout gets a link that points nowhere; for an unregistered plain file, its content spans more than one line. Step 2 repairs a tracked link whose target is a single line plus trailing line breaks; anything spanning several lines is report-only (the user fixes the content by hand, then re-runs the skill).
* `not export-ignored: <dir>/`: appears only once archive excludes are on; the adapter's top directory is missing from the AI tool export-ignore list. Report-only: tell the user to add a `<dir>/ export-ignore` line to that list. The trailing slash matters, since `dir/` patterns match only directory paths.

## Register and repair commands

Register, per `unregistered:` path:

```
tr -d '\r\n' < <path> > <path>.tmp && mv <path>.tmp <path>
git update-index --add --cacheinfo "120000,$(tr -d '\r\n' < <path> | git hash-object -w --stdin),<path>"
```

Repair, per single-line `malformed:` tracked link:

```
[ -L <path> ] || { tr -d '\r\n' < <path> > <path>.tmp && mv <path>.tmp <path>; }
git update-index --cacheinfo "120000,$(git cat-file -p :<path> | tr -d '\r\n' | git hash-object -w --stdin),<path>"
```

## Why the register and repair commands look the way they do

**Register** (`unregistered:` paths). The first command strips line breaks from the plain file itself: with symlinks off, a later `git add` that re-hashes that file stores its bytes as the link target, so a break left on disk would come back. The second stages the file's content as a symlink target (mode `120000`), which is exactly what a symlink-aware checkout would have produced; the plain file on disk then reports as `broken` until symlink support is on. On **skip**, the next `git add` of that path commits it as a regular file.

**Repair** (single-line `malformed:` tracked links). The first command strips the same breaks from a plain file on disk, for the reason above, and leaves a real symlink alone. The second rewrites the staged target without its line breaks. On a symlink-aware checkout the link on disk then reports as `broken`, and `rm <path> && git checkout -- <path>` restores it.

**Broken links** stay report-only: symlink support is a machine setting (on Windows: Developer Mode, then `git config core.symlinks true` in this checkout, or `git config --global core.symlinks true` before a fresh clone), not something this skill changes. Tell the user that until it is on, tools reading through the link see nothing, and that once it is on, `rm <path> && git checkout -- <path>` restores the link.
