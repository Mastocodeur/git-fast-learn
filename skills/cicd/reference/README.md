# CI/CD reference

The golden rules live in the parent [SKILL.md](../SKILL.md). This folder holds the platform- and
cloud-specific detail, factored on **two orthogonal axes**:

```
reference/
  ci/          pipeline + merge gate + secret store + wiki, per CI platform (gitlab, azure-devops)
  cloud/       deploy step + secret injection, per cloud target (azure, gcp)
  templates/   ready-to-copy pipelines — replace <PLACEHOLDER>, never add secrets
```

## How the pieces fit

Setting up a repo = **one CI platform × one cloud target**. Pick one `ci/` file and one `cloud/`
file, then drop the cloud's deploy step into the CI skeleton's `deploy` stage.

The **CI axis owns orchestration** (merge gate, secret store, docs sync); the **cloud axis owns
the deploy step + secret injection**. Keeping them separate means a repo can switch cloud without
rewriting the pipeline, and switch CI without rewriting the deploy recipe.
