# Failure modes

Loaded from `hf-upload/SKILL.md`. Read this file when an upload step fails. The Hard rules in `SKILL.md` remain in force throughout.

| Error | Cause | Action |
|---|---|---|
| `HfHubHTTPError 401` | Token missing, expired, or revoked | `hf auth login --force`; verify with the `whoami` probe in the Prerequisites section of `SKILL.md` |
| `HfHubHTTPError 403` | Token scope lacks Write on this repo | Token is Read-only or scoped elsewhere; create a Write token at `https://huggingface.co/settings/tokens` |
| `HfHubHTTPError 404` | `repo_id` does not exist, or is private and the token has no access | Verify spelling and visibility; check org membership via the `whoami` probe in the Prerequisites section of `SKILL.md` |
| `HfHubHTTPError 413` | Single file exceeds HF's per-file hard limit (500 GB as of this writing; HF recommends under 200 GB per file, see `https://huggingface.co/docs/hub/storage-limits`) | Split or compress; no workaround above the per-file limit |
| Timeout on a commit with many operations | More than roughly 100 ops in one `create_commit`; the server may still complete it after the client times out | Check the repo's commit list before retrying; split into commits of about 100 ops, or use `HfApi().upload_large_folder` for a whole folder |
| LFS storage quota exceeded | Owner's private storage past the free tier (100 GB as of this writing) | Upgrade the plan or delete unused LFS objects via Settings -> Storage |
| `HfHubHTTPError` on a commit that passed `parent_commit` | The branch moved after the step-2 listing | Rerun step 2 and the step-5 gate; never drop `parent_commit` to force it through |
| `FileNotFoundError` on a local path | Wrong local path | Check `path_or_fileobj`; absolute paths are safest |

All `HfHubHTTPError` instances expose `.response.status_code` and a body; surface both verbatim before suggesting a fix.
