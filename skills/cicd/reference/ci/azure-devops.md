# CI platform — Azure DevOps

Pipeline file: `azure-pipelines.yml` (or `ops/pipelines/azure-pipeline.yaml`). Point the Azure
DevOps pipeline at it: *Pipelines → Edit → Settings → YAML path*. Worked examples to copy:
[`../templates/azure-devops/`](../templates/azure-devops/) (Python Function App, Node/Next.js Web App).

Shape: **CI stage → deploy stage → wiki stage**, one pipeline for all environments.

## Merge gate (never merge on red)

The CI stage must pass **before** a PR can complete. A YAML `trigger` only runs the pipeline
*after* a merge, and Azure Repos ignores the YAML `pr:` block — so the gate is a **Branch Policy
Build Validation**, configured in the UI (not versionable):

*Project Settings → Repositories → `<repo>` → Policies → `<branch>` → Build validation → +* —
pick the pipeline, empty path filter, Trigger **Automatic**, Requirement **Required**, expiration
**Immediately when the branch is updated**. Add one per protected branch (`dev`, `uat`,
`main`/`master`).

The same pipeline runs in two contexts, so **guard deploy/wiki** to run only on a real push:

```yaml
condition: and(succeeded(), ne(variables['Build.Reason'], 'PullRequest'))
```

- **In a PR** → only the CI stage runs → serves as the gate; nothing deploys.
- **On merge** → CI + deploy + wiki run.

Ship the pipeline with this condition **before** enabling the policy, or the first PR build would
deploy non-merged code.

## Secret store

Create variables in *Pipeline → Variables* (or a **Variable group**, optionally backed by Azure
Key Vault); mark secrets **"Keep this value secret"**. Reference as `$(NAME)`; the deploy task
injects them into the service (see `../cloud/<provider>.md`). Non-secret values may live in
`config/<env>`; secrets never.

## One pipeline, N environments

A single pipeline lists every env branch under `trigger` and routes each to its
`config/<env>.yaml` via `${{ if }}`. `config/<env>.yaml` holds **only** that env's target service
(subscription / app / resource group). Shared non-secret values (runtime version, wiki context)
live once in the pipeline. Add an env = add its branch + one `config/<env>.yaml`.

## Wiki stage

Provisions the project wiki if absent, clones it with a PAT, rsyncs `docs/`, and pushes only on a
real content diff. Non-secret context (`ORGA`, `PROJECT`, `REPO`, `GIT_USER_*`) is inline in the
stage `env:`; the only secret is `WIKI_TOKEN`.

> **PAT reminder — always deliver this after setup.** Generate a PAT (*User settings → Personal
> access tokens → New Token*) with scopes **Code (Read & Write)** *and* **Wiki (Read & Write)**,
> then add it as a **secret** pipeline variable `WIKI_TOKEN`. Code R&W is needed to clone/push the
> wiki repo; Wiki R&W to list/create the wiki via REST. Without both, the stage fails 401/403.

Page/navigation conventions (`.order`, `/page-name` links) are owned by `docs-readme-wiki`.

## Setup checklist to hand the user

- Branch Policy Build Validation (Required) on each protected branch.
- Secret pipeline variables created for every app secret, plus `WIKI_TOKEN`.
- ARM **service connection** (`azureSubscription`) exists; agent pool / `vmImage` correct.
