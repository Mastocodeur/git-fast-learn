# Pre-commit : Maîtriser ses hooks

> Pre-commit n'est pas un obstacle. C'est un filet de sécurité que **tu configures toi-même**. Ce guide explique comment le plier à tes besoins pour qu'il travaille pour toi, pas contre toi.

---

## Table des matières

1. [Le principe](#1-le-principe)
2. [Le flux d'exécution](#2-le-flux-dexécution)
3. [Installation](#3-installation)
4. [Structure du `.pre-commit-config.yaml`](#4-structure-du-pre-commit-configyaml)
5. [Les clés de configuration par niveau](#5-les-clés-de-configuration-par-niveau)
6. [Catalogue des hooks](#6-catalogue-des-hooks)
7. [Contrôler le périmètre d'un hook](#7-contrôler-le-périmètre-dun-hook)
8. [Les stages : quand un hook tourne](#8-les-stages--quand-un-hook-tourne)
9. [Hooks locaux : créer les siens](#9-hooks-locaux--créer-les-siens)
10. [Reprendre le contrôle au quotidien](#10-reprendre-le-contrôle-au-quotidien)
11. [Config recommandée projet Python](#11-config-recommandée-projet-python)
12. [Anti-patterns à éviter](#12-anti-patterns-à-éviter)

---

## 1. Le principe

Pre-commit intercepte des événements git (commit, push, merge) et exécute des scripts de vérification appelés **hooks**. Chaque hook tourne dans un environnement isolé — rien à installer globalement.

**Tu choisis** quels hooks activer, sur quels fichiers, avec quels arguments, à quel moment. Tout se configure dans `.pre-commit-config.yaml`.

---

## 2. Le flux d'exécution

```
toi: git add fichier.py
toi: git commit -m "feat: truc"
     │
     ▼
pre-commit s'active
     │
     ├─ Hook 1: trailing-whitespace ──► corrige ──► re-stage auto
     ├─ Hook 2: ruff --fix           ──► corrige ──► re-stage auto
     ├─ Hook 3: ruff-format          ──► corrige ──► re-stage auto
     │
     ▼
  Tous OK ? ──► commit passe
  Un KO ?   ──► commit bloqué, tu corriges, tu recommences
```

**Point clé** : les hooks qui ont l'option `--fix` ou qui sont des formatters **modifient les fichiers automatiquement**. Après un échec, souvent il suffit de `git add` + `git commit` à nouveau — le fix est déjà fait.

---

## 3. Installation

```bash
# Avec uv
uv add --dev pre-commit
uv run pre-commit install

# Avec pip
pip install pre-commit
pre-commit install

# Installer aussi le hook commit-msg (pour commitizen par ex.)
pre-commit install --hook-type commit-msg
```

---

## 4. Structure du `.pre-commit-config.yaml`

```yaml
minimum_pre_commit_version: "4.0.0"   # ← Clé GLOBALE
fail_fast: false                        # ← Clé GLOBALE

repos:                                  # ← Liste de REPOS
  - repo: https://github.com/...       # ← Clé REPO
    rev: v5.0.0                         # ← Clé REPO
    hooks:                              # ← Liste de HOOKS
      - id: trailing-whitespace         # ← Clé HOOK
        args: [--fix]                   # ← Clé HOOK
        files: ^src/                    # ← Clé HOOK
        exclude: ^src/generated/        # ← Clé HOOK
```

3 niveaux de config : **global** → **repo** → **hook**. Plus on descend, plus c'est spécifique.

---

## 5. Les clés de configuration par niveau

### Niveau global

| Clé | Type | Défaut | Description |
|-----|------|--------|-------------|
| `minimum_pre_commit_version` | string | aucun | Version min requise. Bloque si version installée trop basse |
| `fail_fast` | bool | `false` | `true` : arrête au premier hook KO. `false` : exécute tout et affiche tous les échecs |
| `default_language_version` | map | — | Version par défaut des langages. Ex: `python: python3.12` |
| `default_stages` | list | `[pre-commit]` | Stages par défaut si le hook n'en spécifie pas |

### Niveau repo

| Clé | Type | Description |
|-----|------|-------------|
| `repo` | string | URL du repo git **ou** valeur spéciale : `local` (hooks du projet), `meta` (hooks internes pre-commit) |
| `rev` | string | Tag git ou SHA. **Toujours figer.** Mettre à jour avec `pre-commit autoupdate` |
| `hooks` | list | Hooks à activer depuis ce repo |

### Niveau hook

| Clé | Type | Défaut | Rôle | Exemple |
|-----|------|--------|------|---------|
| `id` | string | — | Identifiant du hook (défini par le repo source) | `ruff` |
| `name` | string | = id | Nom affiché dans la sortie | `Ruff linter` |
| `args` | list | `[]` | Arguments CLI passés à l'outil | `[--fix, --select, I]` |
| `files` | regex | `""` (tous) | Fichiers à inclure | `^src/.*\.py$` |
| `exclude` | regex | `""` (aucun) | Fichiers à exclure | `^tests/fixtures/` |
| `types` | list | défini par hook | Types de fichiers (par extension) | `[python]` |
| `types_or` | list | — | Comme `types` mais en OU logique | `[python, pyi]` |
| `stages` | list | `[pre-commit]` | Quand le hook tourne | `[pre-push]` |
| `pass_filenames` | bool | `true` | Passe les noms de fichiers staged au hook | `false` pour hooks globaux |
| `always_run` | bool | `false` | Tourne même si aucun fichier ne matche | `true` pour checks globaux |
| `verbose` | bool | `false` | Affiche la sortie même si le hook passe | `true` pour debug |
| `language` | string | auto | Langage du hook | `python`, `node`, `system` |
| `additional_dependencies` | list | `[]` | Dépendances extra à installer dans l'env du hook | `[mypy-extensions]` |

---

## 6. Catalogue des hooks

### 6.1 Hooks génériques — `pre-commit/pre-commit-hooks`

Repo : `https://github.com/pre-commit/pre-commit-hooks`

| Hook ID | Ce qu'il fait | Auto-fix ? | Args utiles | Recommandation |
|---------|--------------|:----------:|-------------|:--------------:|
| `trailing-whitespace` | Supprime espaces/tabs en fin de ligne | Oui | `--markdown-linebreak-ext=md` (préserve les `  ` en markdown) | **Toujours** |
| `end-of-file-fixer` | Ajoute un newline final si manquant | Oui | — | **Toujours** |
| `check-yaml` | Valide la syntaxe YAML | Non | `--allow-multiple-documents`, `--unsafe` (pour tags custom) | **Toujours** |
| `check-toml` | Valide la syntaxe TOML | Non | — | **Toujours** (pyproject.toml) |
| `check-json` | Valide la syntaxe JSON | Non | — | Si JSON dans le projet |
| `check-xml` | Valide la syntaxe XML | Non | — | Si XML dans le projet |
| `check-added-large-files` | Bloque fichiers au-dessus d'un seuil | Non | `--maxkb=500` (défaut 500) | **Toujours** |
| `check-merge-conflict` | Détecte marqueurs `<<<<<<<` / `>>>>>>>` | Non | — | **Toujours** |
| `check-case-conflict` | Détecte fichiers ne différant que par la casse (problème Windows) | Non | — | **Toujours** |
| `detect-private-key` | Détecte clés privées (RSA, DSA, EC, etc.) | Non | — | **Toujours** |
| `check-ast` | Vérifie que le Python parse sans erreur | Non | — | **Toujours** (Python) |
| `debug-statements` | Détecte `breakpoint()`, `import pdb`, `import ipdb` | Non | — | **Toujours** (Python) |
| `no-commit-to-branch` | Bloque commit sur certaines branches | Non | `--branch=main`, `--branch=master` | Selon workflow |
| `mixed-line-ending` | Détecte mix CRLF/LF dans un même fichier | Oui | `--fix=lf` ou `--fix=crlf` | Projets cross-platform |
| `check-symlinks` | Vérifie que les symlinks pointent vers un fichier existant | Non | — | Si symlinks utilisés |
| `check-executables-have-shebangs` | Vérifie qu'un fichier exécutable a un shebang | Non | — | Projets avec scripts |
| `name-tests-test` | Vérifie que les fichiers de test commencent par `test_` | Non | `--pytest-test-first` | Selon convention |
| `pretty-format-json` | Reformate le JSON avec indentation | Oui | `--autofix`, `--indent=2` | Si JSON committé souvent |

### 6.2 Linters et formatters Python

| Outil | Repo | Hook ID | Rôle | Auto-fix ? | Args clés | Quand choisir |
|-------|------|---------|------|:----------:|-----------|---------------|
| **Ruff** | `astral-sh/ruff-pre-commit` | `ruff` | Lint (remplace flake8, isort, pyflakes, bandit partiellement) | Oui | `--fix`, `--select=I` (imports only), `--ignore=E501` | **Défaut recommandé** — 10-100x plus rapide que flake8 |
| **Ruff** | `astral-sh/ruff-pre-commit` | `ruff-format` | Formatage (remplace black) | Oui | `--line-length=120` | **Défaut recommandé** — cohérent avec ruff lint |
| **Black** | `psf/black` | `black` | Formatage | Oui | `--line-length=120` | Seulement si projet existant déjà sur black |
| **isort** | `pycqa/isort` | `isort` | Tri des imports | Oui | `--profile=black` | **Inutile avec ruff** (intégré) |
| **Flake8** | `pycqa/flake8` | `flake8` | Lint | Non | `--max-line-length=120` | **Inutile avec ruff** (intégré) |
| **MyPy** | `pre-commit/mirrors-mypy` | `mypy` | Type checking statique | Non | `--strict`, `--ignore-missing-imports` | Si typage strict. **Lent** — souvent mieux en CI |
| **Bandit** | `pycqa/bandit` | `bandit` | Audit sécurité | Non | `-ll` (medium+), `--skip=B101` | Audit sécu. Ruff couvre une partie via `--select=S` |
| **Pylint** | `pylint-dev/pylint` | `pylint` | Lint avancé | Non | `--disable=C0114` | Lourd — préférer ruff |
| **pyupgrade** | `asottile/pyupgrade` | `pyupgrade` | Modernise la syntaxe Python | Oui | `--py312-plus` | Pour migrer vers syntaxe moderne |

### 6.3 Autres langages et outils

| Outil | Repo | Hook ID | Rôle | Auto-fix ? |
|-------|------|---------|------|:----------:|
| **Prettier** | `pre-commit/mirrors-prettier` | `prettier` | Format JS/TS/CSS/HTML/MD/YAML | Oui |
| **ESLint** | `pre-commit/mirrors-eslint` | `eslint` | Lint JS/TS | Oui (`--fix`) |
| **ShellCheck** | `shellcheck-py/shellcheck-py` | `shellcheck` | Lint Bash/Shell | Non |
| **Hadolint** | `hadolint/hadolint` | `hadolint` | Lint Dockerfiles | Non |
| **SQLFluff** | `sqlfluff/sqlfluff` | `sqlfluff-lint` | Lint SQL | Non |
| **commitizen** | `commitizen-tools/commitizen` | `commitizen` | Enforce conventional commits | Non |
| **codespell** | `codespell-project/codespell` | `codespell` | Détecte fautes de frappe dans le code | Oui |

---

## 7. Contrôler le périmètre d'un hook

C'est **la clé pour ne pas subir pre-commit**. Chaque hook peut être restreint à exactement ce que tu veux.

### 7.1 Par fichiers (`files` / `exclude`)

```yaml
- id: ruff
  args: [--fix]
  files: ^src/            # ne lint QUE src/
  exclude: ^src/legacy/   # sauf le code legacy
```

| Objectif | Config |
|----------|--------|
| Lint seulement `src/` | `files: ^src/` |
| Exclure les fixtures de test | `exclude: ^tests/fixtures/` |
| Seulement les fichiers Python dans `app/` | `files: ^app/.*\.py$` |
| Tout sauf les migrations | `exclude: /migrations/` |
| Seulement un fichier précis | `files: ^pyproject\.toml$` |

### 7.2 Par type de fichier (`types` / `types_or`)

```yaml
- id: trailing-whitespace
  types: [python]          # seulement les .py
```

| Type | Extensions matchées |
|------|-------------------|
| `python` | `.py` |
| `yaml` | `.yml`, `.yaml` |
| `json` | `.json` |
| `toml` | `.toml` |
| `markdown` | `.md` |
| `bash` | `.sh`, `.bash` |
| `javascript` | `.js` |
| `typescript` | `.ts` |
| `sql` | `.sql` |
| `text` | `.txt` |

### 7.3 Par stage (`stages`)

Voir section suivante.

---

## 8. Les stages : quand un hook tourne

Par défaut tout tourne au `pre-commit` (avant commit). Mais tu peux décaler des hooks lourds au push.

| Stage | Quand | Cas d'usage |
|-------|-------|-------------|
| `pre-commit` | Avant `git commit` | Formatage, lint rapide, whitespace — **le plus courant** |
| `pre-push` | Avant `git push` | Tests, type checking, checks lents |
| `commit-msg` | Après saisie du message | Validation format du message (conventional commits) |
| `pre-merge-commit` | Avant commit de merge | Checks spécifiques au merge |
| `manual` | Jamais auto, seulement `pre-commit run --hook-stage manual` | Hooks lourds à lancer à la demande |

```yaml
# Installer le hook pre-push (une seule fois)
# pre-commit install --hook-type pre-push

- id: mypy
  stages: [pre-push]      # trop lent pour chaque commit, tourne au push

- id: commitizen
  stages: [commit-msg]    # valide le format du message
```

**Stratégie recommandée** :

| Vitesse | Stage | Exemples |
|---------|-------|----------|
| < 1s | `pre-commit` | whitespace, ruff, ruff-format, check-yaml |
| 1-10s | `pre-commit` | OK si pas trop de fichiers |
| > 10s | `pre-push` ou CI | mypy, pytest, pylint |

---

## 9. Hooks locaux : créer les siens

Pour lancer un script ou une commande du projet sans dépendre d'un repo externe.

```yaml
- repo: local
  hooks:
    - id: check-env-example
      name: Vérifier que .env.example est à jour
      entry: python scripts/check_env.py
      language: python
      pass_filenames: false
      always_run: true

    - id: no-print
      name: Pas de print() en prod
      entry: grep -n "print("
      language: system
      types: [python]
      files: ^src/
      exclude: ^src/debug/
```

| Clé | Pourquoi |
|-----|----------|
| `language: system` | Utilise un binaire déjà installé sur la machine |
| `language: python` | Crée un venv isolé pour le script |
| `pass_filenames: false` | Le hook tourne une seule fois, pas une fois par fichier |
| `always_run: true` | Tourne même si aucun fichier staged ne matche le filtre |

---

## 10. Reprendre le contrôle au quotidien

### 10.1 Commandes essentielles

| Commande | Quand l'utiliser |
|----------|-----------------|
| `pre-commit run --all-files` | Vérifier tout le repo d'un coup (CI, onboarding) |
| `pre-commit run ruff --all-files` | Lancer un seul hook sur tout le repo |
| `pre-commit run --files src/main.py` | Lancer les hooks sur un fichier précis |
| `pre-commit autoupdate` | Mettre à jour toutes les `rev` vers la dernière version |
| `pre-commit clean` | Vider le cache (si un hook semble buggé) |
| `pre-commit gc` | Nettoyer les anciens environnements de hooks |

### 10.2 Bypass ponctuel

| Méthode | Commande | Quand |
|---------|----------|-------|
| Skip tous les hooks | `git commit --no-verify` | Urgence, WIP |
| Skip un hook | `SKIP=ruff git commit -m "wip"` | Un hook bloque mais pas les autres |
| Skip plusieurs hooks | `SKIP=ruff,ruff-format git commit -m "wip"` | Idem, plusieurs hooks |

> **Règle** : `--no-verify` = dette technique. Chaque usage devrait être suivi d'un `pre-commit run --all-files` avant le push.

### 10.3 Quand un hook échoue

```
ruff.....................................................................Failed
- hook id: ruff
- exit code: 1
- files were modified by this hook
```

**3 cas possibles :**

| Cas | Ce qui se passe | Action |
|-----|----------------|--------|
| Hook auto-fix (ex: ruff --fix) | Le fichier est déjà corrigé | `git add .` → `git commit` à nouveau |
| Hook sans fix (ex: mypy) | Erreur affichée | Corriger manuellement → `git add` → `git commit` |
| Faux positif | Le hook se trompe | Ajouter une exception inline (`# noqa`, `# type: ignore`) ou ajuster `exclude` |

### 10.4 Exceptions inline par outil

| Outil | Ignorer une ligne | Ignorer une règle spécifique |
|-------|-------------------|------------------------------|
| Ruff | `# noqa` | `# noqa: E501` |
| MyPy | `# type: ignore` | `# type: ignore[attr-defined]` |
| Black/Ruff-format | `# fmt: off` / `# fmt: on` (bloc) | — |
| Bandit | `# nosec` | `# nosec B101` |
| Pylint | `# pylint: disable=C0114` | Idem |

---

## 11. Config recommandée projet Python

```yaml
minimum_pre_commit_version: "4.0.0"
fail_fast: false

repos:
  # --- Hooks génériques (rapides, universels) ---
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v5.0.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-toml
      - id: check-added-large-files
        args: [--maxkb=500]
      - id: check-merge-conflict
      - id: check-case-conflict
      - id: detect-private-key
      - id: debug-statements
      - id: check-ast

  # --- Lint + Format Python (rapide) ---
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.11.13
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format
```

**Ordre important** : hooks génériques d'abord (rapides, fixent le whitespace) → lint avec fix → format en dernier.

---

## 12. Anti-patterns à éviter

| Anti-pattern | Pourquoi c'est un problème | Solution |
|--------------|---------------------------|----------|
| Mettre mypy/pytest en `pre-commit` | Trop lent, frustre les devs, pousse au `--no-verify` | Les mettre en `pre-push` ou CI |
| Ne pas figer les `rev` | Résultat différent selon les machines, builds non reproductibles | Toujours utiliser un tag, jamais `main` ou `HEAD` |
| `fail_fast: true` en config partagée | Les devs ne voient qu'une erreur à la fois | Laisser `false`, chacun peut overrider localement |
| Trop de hooks dès le départ | Overwhelm l'équipe, hooks désactivés en masse | Commencer minimal, ajouter progressivement |
| `--no-verify` systématique | Aucune valeur apportée par pre-commit | Réduire les hooks au strict nécessaire |
| Pas de `pre-commit autoupdate` | Versions figées vieillissent, bugs connus non corrigés | Planifier un `autoupdate` mensuel |
| Hook custom sans `pass_filenames: false` | Le script reçoit la liste des fichiers alors qu'il n'en a pas besoin | Adapter selon le script |
