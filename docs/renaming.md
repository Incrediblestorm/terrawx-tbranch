# Renaming or moving the repos

Repo names and locations appear in as few places as possible. Most of what
depends on them is set in repo settings or detected at run time.

Rename on GitHub (repo **Settings → General → Repository name**, or
`gh repo rename`). GitHub keeps secrets, variables, environments and rulesets,
and redirects the old URLs, but don't rely on the redirects: update the
places below.

## The central repo

| Where | What to change |
|---|---|
| Each project repo: **Settings → Secrets and variables → Actions → Variables** | `CENTRAL_AWX_REPO` = new `owner/name` |
| `awx-infra` `terraform.tfvars`: `github_repos` | the new URL, then `tofu apply` (re-registers the runners) |
| This repo | `git submodule set-url central-awx <new-url>` |

Nothing inside the central repo refers to its own name or URL. Codegen reads
the name at run time (for the `managed-by:` label, which follows on the next
run) and points project modules at its own checkout. The dispatch token is a
fine-grained PAT, which is tied to the repo, not its name.

## A project repo

| Where | What to change |
|---|---|
| Central `projects.auto.tfvars.json` | that project's `repo_url` |
| This repo | `git submodule set-url <path> <new-url>` |

The AWX names come from the project's `name` in `projects.auto.tfvars.json`,
not from its repo, so a repo rename changes nothing in AWX (the AWX projects'
SCM URLs are updated in place). Changing `name` itself renames everything in
AWX and recreates the templates, so their job history is lost.

## This repo (the parent)

Rename it; nothing refers to it.

## Moving to an organization

The same steps, with the new owner in every `owner/name` and URL. With an
organization the runners could also be registered once for the whole
organization instead of per repo.

## Submodule paths

Directory names here are independent of the repo names. To change one:
`git mv <old-path> <new-path>` (updates `.gitmodules` too).
