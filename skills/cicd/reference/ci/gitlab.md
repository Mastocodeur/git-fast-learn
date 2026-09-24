# CI platform — GitLab

Pipeline file: `.gitlab-ci.yml` at the repo root. Generic skeleton to copy:
[`../templates/gitlab/.gitlab-ci.yml`](../templates/gitlab/.gitlab-ci.yml). Full worked example
(GitLab → GCP Cloud Run): [`../templates/gitlab/gcp-cloud-run/`](../templates/gitlab/gcp-cloud-run/).

Shape: **`quality` → `deploy` → `docs`**. `quality` is the merge gate; `deploy` and `docs` run
only after a merge to a protected branch.

## Merge gate (never merge on red)

The `quality` job runs pre-commit + tests on merge-request pipelines. Block the merge until it
passes — this is a **project setting**, not something the YAML can enforce alone:

1. **Protect the branches** — *Settings → Repository → Protected branches*: protect `dev`, `uat`,
   `main`/`master` (Maintainers can merge, no direct push).
2. **Require green pipelines** — *Settings → Merge requests*: enable **"Pipelines must succeed"**
   (and "All threads must be resolved" if wanted). Now an MR can't be merged while `quality` is red.

Gate jobs with `rules` so `quality` runs on MRs and branch pushes, while `deploy`/`docs` run only
on protected branches:

```yaml
quality:
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH

deploy:
  rules:
    - if: $CI_COMMIT_BRANCH =~ /^(dev|uat|main|master)$/
      when: on_success
    - when: never
```

## Secret store

*Settings → CI/CD → Variables* → **Add variable**, tick **Masked** and **Protected** (protected =
exposed only on protected branches/tags). Reference as `$VAR_NAME` in the deploy job; the deploy
step forwards them to the cloud service (see `../cloud/<provider>.md`). Never echo a secret.

## One pipeline, N environments

Route by branch with `environment` + `rules`. Per-env differences (project/app/region) come from
`config/<env>` files loaded in the job, or from per-environment CI/CD variables scoped to the
branch. Keep shared values in the pipeline once.

## Wiki / docs sync

Publish `docs/` to the GitLab **project wiki** (a separate `<project>.wiki.git` repo) with the
`docs` stage — copy [`../templates/gitlab/wiki-sync.gitlab-ci.yml`](../templates/gitlab/wiki-sync.gitlab-ci.yml).
It rsyncs `docs/` into the wiki repo and pushes only on a real diff (idempotent), on the default
branch only.

- Token: **`CI_WIKI_TOKEN`** — a **Project Access Token**, **Developer** role, **`write_repository`**
  scope, stored as a masked CI/CD variable. Required to clone/push the wiki repo.
- Enable **Wiki** and **CI/CD** in *Settings → General → Visibility*.
- Page/navigation conventions (`_sidebar.md`, slug-relative links) are owned by `docs-readme-wiki`.

## Setup checklist to hand the user

- Protected branches + "Pipelines must succeed" enabled.
- CI/CD variables created (masked + protected): every app secret, plus `CI_WIKI_TOKEN`.
- Cloud auth configured — prefer **OIDC / Workload Identity Federation** over long-lived keys
  (see `../cloud/<provider>.md`).
