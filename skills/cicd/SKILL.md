---
name: cicd
description: |
  Set up or maintain CI/CD pipelines following the generic standard: a merge gate (pre-commit +
  branch policy on dev/uat/main/master — never merge on red), secrets injected only via the CD
  (never typed into the cloud service by hand), and clear, simple, minimal pipelines (one
  pipeline / N environments, no dead YAML). Platform- and cloud-specific recipes live in
  reference/ (CI: GitLab, Azure DevOps; cloud: Azure, GCP). Use when the user types /cicd, asks
  to "set up the pipeline", "create the CI/CD", "add a deploy pipeline", "inject secrets", "add
  branch policies / merge gate", "sync the wiki", or is working on .gitlab-ci.yml /
  azure-pipelines*.yml / ops/pipelines / pipelines/ files.
---

# CI/CD (generic standard)

Universal CI/CD practices for every repo. The pipeline is always the same shape —
**CI (merge gate) → deploy → docs** — and stays **clear, simple and minimal**. Platform and
cloud specifics live in `reference/`; combine one CI platform with one cloud target.

## Golden rules — apply to every pipeline

1. **Merge gate — never merge on red.** A merge into any protected branch (`dev`, `uat`, `main`,
   `master`) is blocked until CI passes: pre-commit + tests. Enforced by the platform's **branch
   policy**, not by the pipeline trigger.
2. **Secrets only via the CD.** Every secret is injected into the cloud service **at deploy
   time** from the CI/CD secret store. **Never** type a secret by hand into the cloud console,
   and **never** commit one (no secret in YAML, in `config/<env>`, in git).
3. **Clear, simple, minimal.** One pipeline, N environments; `config/<env>` holds *only* what
   differs per env; everything shared lives once. No copy-paste per env, no dead steps, no clever
   tricks — readable beats short.

## 1. Merge gate — never merge on red

- The **CI stage** (pre-commit hooks + tests) must pass **before** a PR/MR can complete. It runs
  the same hooks locally and in CI — see the `check-quality` skill for the hook set.
- This gate is a **branch policy**, not the YAML `trigger`/`rules` — a trigger only fires the
  pipeline *after* the code lands, so it cannot block a merge. Configure it in the platform UI:
  - **GitLab** — protected branch + *"Pipelines must succeed"* on the MR. See `reference/ci/gitlab.md`.
  - **Azure DevOps** — *Branch Policy → Build Validation (Required)*. See `reference/ci/azure-devops.md`.
- Apply one policy per protected branch you gate (`dev`, `uat`, `main`, `master`).
- **Guard deploy/docs stages** so they never run in a PR/MR build — only after a real merge
  (push to the branch). Otherwise a PR build would deploy non-merged code.

## 2. Secret injection — always via the CD

- Secrets live in the **CI/CD secret store**, masked/protected:
  - GitLab → *Settings → CI/CD → Variables* (Masked + Protected).
  - Azure DevOps → secret pipeline variables / a variable group (optionally backed by Key Vault).
- The **deploy job injects them into the cloud service** at deploy time (app settings, env vars,
  or secret references) — recipe per cloud in `reference/cloud/<provider>.md`.
- **Never** enter a secret in the cloud console by hand, and **never** commit one. Non-secret
  config (resource names, region, runtime version) may live in versioned `config/<env>`; secrets
  never do.
- Add a **pre-flight check** that fails early with a readable message if a required secret
  variable is empty — better than an opaque runtime error.

## 3. Keep it clear, simple, minimal

- **One pipeline, N environments.** A single pipeline triggers on each environment branch and
  routes to the matching `config/<env>`; only the **target service** differs per env
  (subscription / project / app / resource). Adding an env = add its branch + one `config/<env>`.
- **`config/<env>` holds only what changes.** Shared, non-secret values live once in the pipeline.
- **No duplication.** Reuse via `extends`/`include` (GitLab) or templates (Azure DevOps) instead
  of copy-pasting stages per env.
- **No dead YAML.** Every job earns its place; prefer three readable stages (CI → deploy → docs)
  over a maze of conditions.

## 4. Build where you deploy

If the runtime has no registry/internet access (e.g. a private VNet), install dependencies **in
the pipeline** and ship them inside the deploy artifact, and **align the runtime version** to the
build — don't rely on the platform building at deploy time.

## 5. Documentation sync

A final **docs stage** publishes `docs/` to the project wiki, **idempotently** (push only on a
real content diff). The content/diagram conventions and the sync mechanism per platform are owned
by the `docs-readme-wiki` skill.

## Choose your setup

Combine **one CI platform** + **one cloud target**:

| Axis | Options | Reference |
|---|---|---|
| CI platform | GitLab · Azure DevOps | `reference/ci/gitlab.md` · `reference/ci/azure-devops.md` |
| Cloud target | Azure · GCP | `reference/cloud/azure.md` · `reference/cloud/gcp.md` |
| Ready-to-copy | pipeline skeletons & worked examples | `reference/templates/` |

See [reference/README.md](reference/README.md) for how the pieces fit together.

## Steps to set up a repo

1. **Detect** the CI platform (`.gitlab-ci.yml` vs `azure-pipelines*.yml`) and the cloud target.
2. **Copy** the matching template from `reference/templates/`; replace the `<PLACEHOLDER>` values.
3. **One pipeline, N envs**: list each environment branch, route it to its `config/<env>` holding
   only that env's service. Keep shared non-secret values in the pipeline, once.
4. **Classify** each app setting secret vs non-secret; wire every secret as a CD-injected variable
   from the secret store (never in git).
5. **Enable the merge gate** (branch policy / build validation) on each protected branch so a PR/MR
   can't merge until CI passes.
6. **Add the docs/wiki sync** stage (see `docs-readme-wiki`).
7. **Deliver the secret checklist** to the user: the exact variables/tokens they must create in
   the CI/CD secret store, and any cloud auth (service connection / workload identity).
