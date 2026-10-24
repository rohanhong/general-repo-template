# Example: add a weight and bump the manifest

Loaded from `hf-upload/SKILL.md`. Read this file at step 8 when the consumer pins HF revisions in a manifest, or when the user asks for a worked run. The Hard rules in `SKILL.md` remain in force throughout.

This is a worked example of a whole run, not a script to execute from step 8. Its upload (block step 2) is what `SKILL.md` steps 5 and 6 run behind the choice-question gate; from step 8, use only block steps 3 to 5.

Run this block in a POSIX shell: the heredoc is a POSIX-shell construct. `<python>` is the interpreter name from the probe in the Prerequisites section of `SKILL.md`, and the local path reaches Python as an argument (same section). Owner, repo, and package names below are placeholders; substitute the ones resolved in the Procedure.

```bash
# 0. Local path, converted for native Windows Python (no-op without cygpath)
P=/path/to/new_weight.pt
command -v cygpath >/dev/null 2>&1 && P="$(cygpath -m "$P")"

# 1. Hash locally first (the manifest needs it; the step-3 one-liner)
<python> -c 'import hashlib,sys;h=hashlib.sha256();f=open(sys.argv[1],"rb");[h.update(b) for b in iter(lambda:f.read(1<<20),b"")];print(h.hexdigest())' "$P"
# -> abcdef1234... (record this)

# 2. Upload (sys.argv: local path, then the step-2 sha)
<python> - "$P" "<sha>" <<'PY'
import sys
from huggingface_hub import HfApi, CommitOperationAdd
api = HfApi()
info = api.create_commit(
    repo_id="acme/foo-bot-models",
    repo_type="model",
    operations=[
        CommitOperationAdd(
            path_in_repo="new_weight.pt",
            path_or_fileobj=sys.argv[1],
        ),
    ],
    commit_message="add new_weight v1",
    parent_commit=sys.argv[2],
)
print("OID:", info.oid)
print("URL:", info.commit_url)
PY
# -> OID: 4268c2a3b3...
# -> URL: https://huggingface.co/acme/foo-bot-models/commit/4268c2a3...

# 3. Update the consumer manifest (only if the consumer pins HF revisions)
#    Edit <package>/config/model_weights.yaml:
#      hf_revision: 4268c2a3b3190ada3545886d395efba0ffda5f5a
#      weights.<name>.sha256: abcdef1234...

# 4. Verify locally
<python> <package>/scripts/fetch_weights.py
# Expect: downloaded -> ... then sha256 OK

# 5. Commit the yaml change in the consumer git repo (separate workflow)
```
