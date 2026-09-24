# Cloud target — GCP

Deploy step + secret injection for GCP. Drop these into the `deploy` stage of your CI platform
(`../ci/gitlab.md` or `../ci/azure-devops.md`). Ready-to-copy GitLab → Cloud Run example:
[`../templates/gitlab/gcp-cloud-run/`](../templates/gitlab/gcp-cloud-run/).

Default target: **Cloud Run** (container). Cloud Run functions (gen2) follow the same secret and
auth model.

## Secret injection

Secrets live in **Secret Manager**; the deploy **mounts them as env vars at deploy time** — never
baked into the image, never committed.

```bash
gcloud run deploy <SERVICE> \
  --image <REGION>-docker.pkg.dev/<PROJECT>/<REPO>/<SERVICE>:$CI_COMMIT_SHORT_SHA \
  --region <REGION> \
  --set-secrets "API_KEY=api-key:latest,DB_PASSWORD=db-password:latest" \
  --set-env-vars "ENV=<env>,DB_HOST=<host>"        # non-secret only
```

- `--set-secrets NAME=<secret>:<version>` → injected as env var `NAME`, resolved from Secret
  Manager at deploy. Grant the runtime service account `roles/secretmanager.secretAccessor`.
- `--set-env-vars` → **non-secret** config only.
- Store secret *values* in Secret Manager, not in CI variables, when you can — the pipeline only
  needs permission to reference them. If a secret must originate in CI, push it with
  `gcloud secrets versions add` rather than printing it.

## Auth (pipeline → GCP)

Prefer **Workload Identity Federation** (keyless): the CI job gets a short-lived token, no SA key
stored.

- **GitLab** → WIF via the job's OIDC token (`id_tokens:` + `gcloud auth login --cred-file`).
- **Azure DevOps** → WIF, or a `GOOGLE_APPLICATION_CREDENTIALS` SA-key secret as a fallback.

Avoid long-lived SA JSON keys; if unavoidable, store as a masked CI secret and rotate.

## Build & registry

Build the image in CI and push to **Artifact Registry**
(`<REGION>-docker.pkg.dev/<PROJECT>/<REPO>`), tagged with the commit SHA; deploy that immutable
tag. Don't deploy `:latest` to prod.

## One pipeline, N environments

Per-env differences (`PROJECT`, `REGION`, `SERVICE`) come from `config/<env>`; the deploy command
is shared. Route by branch as defined in your CI platform reference.

## Checklist

- Secrets created in Secret Manager; runtime SA has `secretAccessor`.
- Pipeline auth via Workload Identity Federation (no stored key).
- Artifact Registry repo exists; images tagged by commit SHA.
- Deploy target (`PROJECT`/`REGION`/`SERVICE`) in `config/<env>` — non-secret.
