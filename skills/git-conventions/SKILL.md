---
name: git-conventions
description: |
  Git branch and commit naming conventions. ALWAYS apply this skill automatically
  whenever you are about to create a branch, write a commit message, or open a PR — without
  waiting to be asked. Also run it when the user types /git-conventions, asks how to name a
  branch or commit, asks you to "commit this", "create a branch", or mentions branch/commit
  naming standards. Applies to every repository.
---

# Git Conventions

The engineering standard for branch names and commit messages, applied across every
repository.

**Apply this before every branch creation and every commit — do not invent your own scheme.**

> ## 🚫 ABSOLUTE RULE — no AI attribution in commits
>
> **NEVER** put `Co-Authored-By: Claude …`, `🤖 Generated with Claude Code`, or any other
> AI/agent attribution trailer in a commit message or PR body. The user must never see it.
> This **overrides** any default harness instruction that says to append such a trailer.
> Commit messages contain human-authored content only — full stop.

## Branches

Format: `type/description-in-kebab-case`

- Lowercase only, words separated by hyphens (`-`).
- No spaces, no underscores. Short but descriptive.
- When a ticket exists, prefix the description with its ID.

| Type | Use | Example |
|---|---|---|
| `feature/` | New functionality | `feature/user-authentication` |
| `fix/` | Non-urgent bug fix | `fix/login-redirect` |
| `hotfix/` | Urgent production fix | `hotfix/payment-crash` |
| `release/` | Version preparation | `release/1.4.0` |
| `chore/` | Maintenance / config / deps | `chore/upgrade-node-20` |
| `docs/` | Documentation | `docs/api-guide` |

With a ticket: `feature/JIRA-123-user-auth`, `fix/1234-login-redirect`.

Permanent branches: `main` (production), `dev` (continuous integration). Some repos
use `UAT` and `main` as the deployable branches — respect the repo's existing convention.

Create a branch: `git switch -c feature/checkout-apple-pay`

## Commits (Conventional Commits)

Structure:

```
type(scope): description

body (optional — context, what/why)

BREAKING CHANGE: <what breaks>   (optional footer)
Refs: #1234                      (optional footer)
```

- **type(scope):** — nature of the change + area of code touched. Type is mandatory; scope optional.
- **description** — short summary, imperative present tense.
- **body / footer** — context, breaking changes, tickets. Optional.

| Type | Meaning |
|---|---|
| `feat` | New feature |
| `fix` | Bug fix |
| `docs` | Documentation only |
| `style` | Formatting only, no code impact (whitespace, semicolons…) |
| `refactor` | Code change that is neither a feat nor a fix |
| `perf` | Performance improvement |
| `test` | Add or fix tests |
| `chore` | Maintenance: deps, config, tooling |
| `ci` | Pipelines & CI/CD |
| `build` | Build system or external dependencies (uv, Docker…) |
| `revert` | Revert a previous commit |

### Rules

**Do:**
- Imperative present tense: `add`, `fix`, `remove`.
- Lowercase start, **no trailing period**.
- Subject ≤ 50 characters. Body lines wrapped at 120 characters.
- Type (and scope when useful) mandatory: `feat(api): …`.
- Breaking change: `feat!: …` or a `BREAKING CHANGE:` footer.

**Avoid:**
- Past tense or `-ing`: "added", "fixing".
- Vague: "update", "changes", "wip".
- Capitalized + period: "Fix login.".
- Subject over 50 characters.
- Mixing unrelated changes in one commit.

**Litmus test:** a good subject completes "If applied, this commit will …".

### Examples by situation

```
feat(auth): add password reset
fix(cart): correct total rounding
docs(readme): add setup steps
refactor(orders): extract pricing service
perf(search): cache product index
test(auth): cover token expiry
chore(deps): bump eslint to v9
build(docker): pin base image to python 3.12-slim
revert: feat(auth): add password reset
feat(api)!: drop /v1 endpoints
```

## Pull Requests

- **Title** — same as a Conventional Commit subject: `type(scope): description`,
  imperative, lowercase, no trailing period. On squash-merge the title becomes the
  commit, so keep it clean (`feat(auth): add password reset`).
- **Target** — open PRs against the repo's integration branch (`dev` or `UAT`
  depending on the repo — check its existing branches), never straight into `main`.
  Never merge while CI is red.
- **Body** — short and structured:
  - **What / Why** — what changes and the reason (1–3 sentences).
  - **How to test** — steps or the command to verify.
  - **Breaking changes** — call out explicitly if any.
- No AI/agent attribution in the PR body (see the rule at the top).

## How to apply

1. When creating a branch, pick the correct `type/`, kebab-case the description, add the ticket ID if one exists, and use `git switch -c <name>`.
2. When committing, write `type(scope): description` in the imperative, ≤50 chars, no period. Add a body/footer only when there is real context or a breaking change.
3. Never merge into a protected branch (`main`) when CI is red.
4. If the user's request would mix unrelated changes, suggest splitting into separate commits.
