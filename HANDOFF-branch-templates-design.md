# Handoff: per-branch AWX job templates — design decisions

From a design session with Greg (cloud Claude, 2026-10-02/03). This captures
what was decided and proven. It does not replace anything already built here;
reconcile it with the current state of `central-awx/` and
`simple-ansible-project/`.

## Goal

Each Ansible project (e.g. `simple-ansible-project`) gets AWX job templates
created automatically for its topic branches on the **test** AWX server, so
developers can run their branch's automation in isolation. Prod only gets the
stable main-branch templates.

## Ownership split

| Thing | Owned by |
|---|---|
| Template definitions (.tf / data) | The Ansible project itself, in its own branch |
| Terraform engine, shared module, pipeline | `central-awx` |
| AWX Team objects (names only) | `central-awx`, created once per server |
| Who has access (team → role) | The automation's author, granted at the **AWX project** level, not per template |
| Team membership | The AWX server itself (not Terraform) |

Why project-level grants: per-template grants would mean maintaining an
endlessly changing permission matrix. Templates inherit access from their
project.

## Environments / servers

- Two persistent AWX servers: **test** and **prod** (already reflected in
  `variables.tf`).
- Separate state per environment. They are never shared.
- Same team names on both servers, but they're separate objects in separate
  states.
- CI environment (GitHub Actions environments) supplies the server URL and
  token.
- **Test path:** enumerate live branches, generate templates for all of them.
- **Prod path:** main-branch templates only, no branch enumeration.

## Core Terraform structure (proven)

One generated `<project>.tf.json` per Ansible project. Each contains **one**
`module` block that uses `for_each` over that project's live branches:

```
module.project_a["topic/x"]   module.project_a["topic/y"]   module.project_b["topic/z"]
```

- A module `source` must be a static literal (it's resolved at `init`, before
  variables exist). Pin it, e.g. `?ref=v1.0.0`. The branch goes in as an
  input, never into `source`.
- Generate the file with **jq**, not printf/string templating. The printf
  version broke on unescaped quotes inside the `for_each` expression.

```bash
gen() {  # gen <project> <branches...>
  p=$1; shift
  jq -n --arg p "$p" --args '{module:{($p):{
      source:"./modules/awx_branch",
      for_each:("${toset(" + ($ARGS.positional|tojson) + ")}"),
      project:$p, branch:"${each.key}", templates:["deploy","patch"]}}}' "$@" > $p.tf.json
}
```

## Targeting rule (non-negotiable)

A project's pipeline generates only **its own** project's file, so the config
on disk is incomplete relative to the shared test state. Therefore:

- Project pipelines **always** run `tofu apply -target=module.<project>`
  (un-indexed). That covers every branch instance of that project and nothing
  else.
- **Never** run a bare apply from a project pipeline. It would destroy every
  other project's templates.
- Keep a periodic **full reconcile** job that generates all projects and
  applies without `-target`. It's the drift backstop, because targeted runs skip
  the full-graph check.

Verified with OpenTofu 1.8.3 (`terraform_data` stand-ins). After deleting
`project_a`'s `topic/y` and generating only project_a:

```
tofu plan -target=module.project_a  ->  3 to destroy  (only project_a["topic/y"])
tofu plan  (no target)              ->  6 to destroy  (also wipes project_b["topic/z"])
```

## Branch deletion

There's no special delete job. Fire the normal project pipeline on the branch
delete event. It regenerates the project's live branch list, which no longer
contains the deleted branch, and the project-grain targeted apply removes that
branch's templates. You never need a branch-level target.

Make each generation **deterministic** (same branches in, byte-identical JSON
out) so unchanged branches no-op instead of churning while someone is testing.

## Provider gotchas (checked against source)

- `denouche/awx`: `awx_job_template` has **no** `scm_branch`; the branch lives on
  `awx_project.scm_branch`. So in practice it's one AWX project per branch, and
  that branch's templates point at it.
- `TravisStratton/awx` does expose `scm_branch` on templates, but it's honored
  only if the project has `allow_override = true`.
- Still to confirm: which resource your chosen provider offers for the
  team-to-project role grant.

## Open items

1. State backend keying per environment (test vs prod).
2. The branch-enumeration step must **fail the run** on an API error. An empty
   list treated as success would destroy everything in that project.
3. Branch-name filter (prefix) so throwaway branches don't spawn template sets.
4. Confirm the role-grant resource (see above).
5. Module version ownership: someone bumps and tests the pinned `ref`.
