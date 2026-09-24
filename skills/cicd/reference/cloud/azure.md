# Cloud target — Azure

Deploy step + secret injection for Azure. Drop these into the `deploy` stage of your CI platform
(`../ci/gitlab.md` or `../ci/azure-devops.md`). Worked Azure DevOps pipelines:
[`../templates/azure-devops/`](../templates/azure-devops/).

## Secret injection

Secrets become **app settings** on the service, set **at deploy time** from the CI secret store —
never typed in the portal, never committed.

- **Azure DevOps** deploy task: pass `-NAME "$(NAME)"` per secret in the task's `appSettings`.
- **GitLab / gcloud-style**: `az functionapp config appsettings set --name <app> --settings NAME="$NAME"`.
- Or store secrets in **Key Vault** and reference them as an app setting:
  `NAME=@Microsoft.KeyVaultReference(SecretUri=https://<vault>.vault.azure.net/secrets/NAME)` — the
  app resolves it at runtime via its **Managed Identity** (`DefaultAzureCredential`).

Non-secret settings (tenant/client IDs, endpoints, DB host/user, resource names) may live in
`config/<env>`; secrets only in the CI secret store.

## Auth (pipeline → Azure)

- **Azure DevOps** → an **ARM service connection** (`azureSubscription`), ideally workload identity
  federation (no stored key).
- **GitLab** → **Workload Identity Federation / OIDC** to Azure AD, or an `AZURE_CREDENTIALS`
  service-principal secret. Prefer OIDC — no long-lived key.

## Python Function App

Deploy task `AzureFunctionApp@2` (`functionAppLinux`), method `zipDeploy`. Because a private-VNet
Function App has **no PyPI at runtime**, build where you deploy:

- Export prod deps: `uv export --no-hashes --no-dev --no-emit-project -o requirements.txt`.
- Install for the target platform:
  `uv pip install --python-platform x86_64-manylinux_2_28 --target <app>/.python_packages/lib/site-packages`.
- **Align the runtime** to the build Python before deploy:
  `az functionapp config set --linux-fx-version "PYTHON|3.12"` — prevents `ModuleNotFoundError: pydantic_core`.
- Disable platform build: app settings `SCM_DO_BUILD_DURING_DEPLOYMENT=false`, `ENABLE_ORYX_BUILD=false`.
- **Size guardrail**: fail if the zip > 900 MB (run-from-package ~1 GB limit).
- Then all app secrets as `-NAME "$(NAME)"`.

## Node / Next.js Web App

Deploy task `AzureWebApp@1` (`webAppLinux`): build → zip → deploy, with a `StartupCommand`.
Inject secrets as app settings the same way (`-NAME "$(NAME)"` / Key Vault references).

## Checklist

- Deploy target (subscription / app / resource group) in `config/<env>` — non-secret.
- Every secret wired as a CD-injected app setting or a Key Vault reference.
- Runtime version aligned build↔service (Function App).
- Pipeline auth via service connection / OIDC, not a committed key.
