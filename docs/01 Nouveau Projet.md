# 01 — Créer un nouveau projet

← [Retour au sommaire](README.md)

---

## Sommaire

- [Option A — Depuis la plateforme (recommandé)](#option-a--depuis-la-plateforme-recommandé)
- [Option B — Depuis le terminal](#option-b--depuis-le-terminal)
- [Configurer le .gitignore](#configurer-le-gitignore)
- [Mettre en place la structure de branches](#mettre-en-place-la-structure-de-branches)
- [Activer les protections de branches](#activer-les-protections-de-branches)
- [Premiers commits](#premiers-commits)

---

## Option A — Depuis la plateforme (recommandé)

Créer le dépôt directement sur GitHub, GitLab ou Azure DevOps avec un README initial, puis cloner :

```bash
git clone git@github.com:<organisation>/<nom-du-projet>.git
cd <nom-du-projet>
```

Le dépôt est prêt. La branche `main` existe déjà et le remote `origin` est configuré automatiquement.

---

## Option B — Depuis le terminal

Pour initialiser un dépôt local et le connecter à un remote vide (créé au préalable sans README) :

```bash
# 1. Initialiser le dépôt local
git init

# 2. Créer le premier commit (nécessaire pour créer la branche main)
echo "# Mon projet" > README.md
git add README.md
git commit -m "init: initialiser le dépôt"

# 3. Renommer la branche en main si nécessaire
git branch -M main

# 4. Connecter au remote
git remote add origin git@github.com:<organisation>/<nom-du-projet>.git

# 5. Pousser et tracker le remote
git push -u origin main
```

Vérifier que le remote est bien configuré :

```bash
git remote -v
# origin  git@github.com:<organisation>/<nom-du-projet>.git (fetch)
# origin  git@github.com:<organisation>/<nom-du-projet>.git (push)
```

---

## Configurer le .gitignore

Le `.gitignore` doit être créé dès le début — avant tout autre commit. Il évite de versionner des fichiers générés, des secrets ou des artefacts locaux.

```bash
# Créer le .gitignore
touch .gitignore
```

Contenu minimal selon le contexte :

```gitignore
# Environnements virtuels
.venv/
venv/
env/

# Secrets et variables d'environnement
.env
.env.local
*.pem
*.key

# IDE
.vscode/
.idea/
*.swp

# OS
.DS_Store
Thumbs.db

# Build et artefacts
__pycache__/
*.pyc
*.pyo
dist/
build/
*.egg-info/

# Données locales (ne pas versionner les données brutes)
data/raw/
data/interim/
```

Générer un `.gitignore` adapté : [gitignore.io](https://www.toptal.com/developers/gitignore)

Commiter le `.gitignore` immédiatement après l'avoir créé :

```bash
git add .gitignore
git commit -m "chore: ajouter le .gitignore"
```

---

## Mettre en place la structure de branches

Créer la branche `dev` dès le début. Toutes les features partiront de `dev`, pas de `main`.

```bash
# Créer et pousser la branche dev
git checkout -b dev
git push -u origin dev
```

Structure finale :

```
main    ← production, code stable et déployé
dev     ← intégration, reçoit toutes les PR de feature
feature/*, fix/*, ...  ← branches de travail éphémères
```

---

## Activer les protections de branches

Dès que le dépôt est créé, activer les protections sur `main` et `dev` pour qu'aucun commit ne puisse y atterrir directement.

### GitHub

`Settings → Branches → Add branch protection rule`

```
Branch name pattern: main

☑ Require a pull request before merging
  Minimum approvals: 1
☑ Require status checks to pass before merging
☑ Do not allow bypassing the above settings
```

### GitLab

`Settings → Repository → Protected branches`

```
Branch: main
Allowed to merge: Maintainers
Allowed to push: No one
```

### Azure DevOps

`Project Settings → Repositories → [repo] → Policies → main`

```
☑ Require a minimum number of reviewers (1)
☑ Build validation → lier le pipeline CI
☑ Reset votes on new pushes
```

---

## Premiers commits

Convention à respecter dès le premier commit :

```bash
# Structure d'un bon commit
git commit -m "<type>(<scope>): <description impérative courte>"

# Exemples
git commit -m "init: initialiser le dépôt"
git commit -m "chore: ajouter le .gitignore"
git commit -m "docs: ajouter le README"
git commit -m "ci: ajouter le pipeline GitHub Actions"
```

**Règle fondamentale :** ne jamais travailler directement sur `main` ou `dev`. Créer systématiquement une branche de travail avant toute modification.

```bash
git checkout dev
git checkout -b feature/ma-premiere-feature
```
