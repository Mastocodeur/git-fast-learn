# Git — Guide Complet

> Un dépôt de référence pour maîtriser Git : des fondamentaux jusqu'aux scénarios d'équipe avancés, avec les questions d'entretien les plus fréquentes.

---

## Sommaire

| # | Fiche | Sujets couverts |
|---|-------|-----------------|
| 00 | [Configuration initiale](00%20Configuration%20Initiale.md) | git config, identité, éditeur, pull.rebase, SSH, aliases, .gitconfig |
| 01 | [Créer un nouveau projet](01%20Nouveau%20Projet.md) | init, remote, .gitignore, structure de branches, branch protection |
| 02 | [Rejoindre un projet existant](02%20Rejoindre%20un%20Projet.md) | clone, branches, conventions commits, MR/PR, tags, issues |
| 03 | [Git au Quotidien](03%20Git%20au%20Quotidien.md) | workflow standard, rebase vs merge, gestion des conflits |
| 04 | [Pre-commit](04%20Pre-commit.md) | hooks, catalogue, config recommandée, bypass ponctuel |
| 05 | [Collaboration en Équipe](05%20Collaboration%20Equipe.md) | 2 devs / même branche, 15 devs (fetch + rebase), branch policies |
| 06 | [Annuler et Corriger](06%20Annuler%20et%20Corriger.md) | restore, reset (soft/mixed/hard), revert, amend, reflog |
| 07 | [Git Avancé](07%20Git%20Avance.md) | stash, cherry-pick, rebase -i, bisect, log avancé, worktree |
| 08 | [CI/CD](08%20CICD.md) | GitHub Actions, GitLab CI, Azure Pipelines |
| 09 | [Questions d'Entretien](09%20Questions%20Entretien.md) | top 25 questions + réponses + mises en situation |

---

## Quick Reference

### Les 3 zones Git

```
Working Directory → (git add) → Staging Area → (git commit) → Repository local → (git push) → Remote
```

### Workflow standard

```bash
git checkout dev && git pull origin dev     # 0. partir d'un dev à jour
git checkout -b feat/ma-feature            # 1. créer sa branche
# ... coder ...
git add src/mon_fichier.py                 # 2. stager les changements
git commit -m "feat(scope): description"  # 3. commiter
git push origin feat/ma-feature            # 4. pusher
# 5. ouvrir une PR/MR sur GitHub/GitLab
```

### Conventions de commits

```
feat | fix | docs | style | refactor | test | chore | perf | ci | revert
```

Format : `<type>(<scope>): <description au présent impératif, ≤72 car.>`

### Conventions de branches

```
feature/ | fix/ | refactor/ | docs/ | test/ | chore/ | hotfix/ | data/ | experiment/
```

Format : `<type>/<description-kebab-case>`

### Hiérarchie des branches

```
feature/x  ──┐
fix/y        ├──► dev ──► main (→ production)
refactor/z  ──┘
```

---

## Skills Claude Code

Le dossier `skills/` contient des instructions pour Claude Code (invocables via `/nom-du-skill`). Ils encodent les standards d'ingénierie appliqués automatiquement dans tous les dépôts.

| Skill | Déclencheur | Description |
|-------|-------------|-------------|
| `git-conventions` | `/git-conventions` ou avant tout commit/branche | Conventions de branches et commits (Conventional Commits, naming kebab-case, règles PR) |
| `check-quality` | `/check-quality` ou après toute édition Python | Lance pre-commit + pytest sur les fichiers modifiés (Ruff, interrogate, gitleaks, uv) |
| `docs-readme-wiki` | `/docs-readme-wiki` ou après tout changement fonctionnel | Maintient README et Wiki synchronisés avec le code, Mermaid pour tous les diagrammes |
| `cicd` | `/cicd` ou sur les fichiers de pipeline | Pipelines CI/CD (GitLab CI, Azure DevOps) — merge gate, secrets par CD, templates dev/uat/prod |
| `python-style` | `/python-style` ou sur toute édition Python | Lisibilité Python : espacement, imports, classes vs fonctions — complémentaire à Ruff |

> Ces skills sont des standards d'ingénierie personnels, applicables à tout projet Python / Azure / GitLab.

---

## Scénarios rapides

| Situation | Procédure |
|-----------|-----------|
| Collègue a pushé sur la même branche | `git stash` → `git pull` → `git stash pop` |
| Mettre à jour sa branche sur une grande équipe | `git fetch origin` → `git rebase origin/dev` |
| Pousser après un rebase | `git push --force-with-lease` |
| Annuler le dernier commit (garder les modifs) | `git reset --soft HEAD~1` |
| Annuler un commit déjà pushé | `git revert <sha>` |
| Retrouver un commit perdu | `git reflog` |
| Intégrer un commit isolé d'une autre branche | `git cherry-pick <sha>` |
| Trouver le commit qui a introduit un bug | `git bisect start` |
| Voir l'historique en graphe | `git log --oneline --graph --all` |
