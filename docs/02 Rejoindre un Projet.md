# 02 — Rejoindre un projet existant

← [Retour au sommaire](README.md)

---

## Sommaire

- [Cloner le dépôt](#cloner-le-dépôt)
- [Créer une branche de travail](#créer-une-branche-de-travail)
- [Branches d'intégration et hiérarchie](#branches-dintégration-et-hiérarchie)
- [Durée de vie des branches](#durée-de-vie-des-branches)
- [Écrire des messages de commit](#écrire-des-messages-de-commit)
- [Tags et versions](#tags-et-versions)
- [Merge Requests / Pull Requests](#merge-requests--pull-requests)
- [Issues](#issues)

---

## Cloner le dépôt

```bash
git clone git@github.com:<organisation>/<nom-du-projet>.git
cd <nom-du-projet>
```

Vérifier l'état du dépôt après le clone :

```bash
git branch -a       # voir toutes les branches (locales + distantes)
git log --oneline   # voir l'historique des commits
git remote -v       # vérifier le remote configuré
```

---

## Créer une branche de travail

**Règle fondamentale : on ne travaille jamais directement sur `main` ou `dev`.** Avant toute modification, créer une branche dédiée depuis `dev` (ou `main` si le projet n'a pas de `dev`) :

```bash
# Partir d'un dev à jour
git checkout dev
git pull origin dev

# Créer sa branche
git checkout -b <type>/<description-courte>
```

Le nom de la branche suit le format `<type>/<description-en-kebab-case>` :

| Type | Usage | Quand l'utiliser | Exemple |
|------|-------|------------------|---------|
| `feature/` | Nouvelle fonctionnalité | On ajoute un comportement qui n'existait pas | `feature/add-data-loader` |
| `fix/` | Correction de bug | Le code existant produit une erreur ou un résultat incorrect | `fix/missing-null-check` |
| `refactor/` | Restructuration | On réorganise sans changer le comportement visible | `refactor/simplify-pipeline` |
| `docs/` | Documentation | Fichiers de doc uniquement, sans toucher au code | `docs/update-readme` |
| `test/` | Tests | Ajout ou correction de tests, sans modifier le code de prod | `test/add-loader-tests` |
| `chore/` | Tâche technique | Mise à jour de dépendances, config CI, outils de dev | `chore/upgrade-ruff` |
| `hotfix/` | Correction urgente | Bug critique en production à corriger immédiatement | `hotfix/fix-prod-crash` |
| `data/` | Données ou pipelines | Pipeline de données, schéma, ingestion, preprocessing | `data/add-preprocessing-step` |
| `experiment/` | Exploration | Test d'une idée ou d'un modèle, pas forcément destiné à merger | `experiment/test-new-model` |

---

## Branches d'intégration et hiérarchie

Les branches de travail (`feature/`, `fix/`…) sont éphémères. Elles s'intègrent dans une hiérarchie de branches stables :

```
feature/ma-feature  ──┐
fix/mon-bug           ├──► dev ──► main (→ production)
refactor/mon-refacto ──┘
```

| Branche | Rôle | Qui y commit |
|---------|------|--------------|
| `feature/*`, `fix/*`… | Travail en cours — durée courte | Chaque développeur sur sa propre branche |
| `dev` | Intégration — reçoit toutes les PR de travail | Personne directement (tout passe par PR) |
| `main` | Production — code stable et déployé | Alimentée uniquement depuis `dev` via PR |

**Le flux complet :**

1. `git checkout -b feature/ma-tache` depuis `dev`
2. Travailler, commiter, pousser
3. Ouvrir une PR de `feature/ma-tache` → `dev`
4. La PR est relue et mergée dans `dev`
5. Quand `dev` est validé, PR de `dev` → `main`
6. Le merge dans `main` déclenche le déploiement via CI/CD

**Récupérer les dernières modifications de `dev` sur sa branche :**

```bash
git fetch origin
git rebase origin/dev
```

---

## Durée de vie des branches

**Une branche qui traîne devient un problème.** Plus une branche vit longtemps, plus elle s'éloigne de `main` et plus le merge sera difficile.

**Règle : une branche de travail ne devrait pas dépasser 3 à 5 jours.** Si c'est le cas, le périmètre est trop large — il faut découper.

```bash
# Supprimer une branche locale après merge
git branch -d feature/ma-tache

# Supprimer la branche distante
git push origin --delete feature/ma-tache

# Voir les branches déjà mergées (candidates à la suppression)
git branch --merged dev
```

**Exception :** les branches `experiment/` ne sont pas soumises à cette règle.

---

## Écrire des messages de commit

**Règle : un commit par changement logique cohérent, avec un message court et précis.**

Format :

```
<type>(<scope>): <description courte au présent impératif>
```

| Type | Quand l'utiliser | Exemple |
|------|-----------------|---------|
| `feat` | Nouvelle fonctionnalité (comportement métier) | `feat(auth): ajouter l'authentification OAuth2` |
| `fix` | Correction de bug | `fix(pipeline): corriger le calcul sur les nulls` |
| `docs` | Documentation uniquement | `docs(readme): mettre à jour l'installation` |
| `style` | Formatage, espaces — sans changement logique | `style: ruff format` |
| `refactor` | Restructuration sans ajout ni correction | `refactor(loader): extraire la normalisation` |
| `test` | Ajout ou correction de tests | `test(loader): ajouter les tests unitaires` |
| `chore` | Maintenance (dépendances, CI, config) | `chore(deps): mettre à jour pandas 2.2.0` |
| `perf` | Amélioration des performances | `perf(query): optimiser le batch processing` |
| `ci` | Pipelines CI/CD | `ci: ajouter le lint step` |
| `revert` | Retour arrière sur un commit | `revert: annuler le changement de config` |

**Bonnes pratiques :**
- Description en anglais, au présent impératif, sans majuscule initiale ni point final
- Moins de 72 caractères
- Un commit = un changement logique (jamais de commits fourre-tout)

---

## Tags et versions

Les tags marquent les versions stables. Convention : **versionnement sémantique** `vX.Y.Z`

| Composant | Signification |
|-----------|---------------|
| `X` (majeur) | Changement incompatible avec la version précédente |
| `Y` (mineur) | Nouvelle fonctionnalité rétrocompatible |
| `Z` (patch) | Correction de bug |

```bash
# Créer un tag annoté (recommandé)
git tag -a v1.0.0 -m "Version 1.0.0 - première release stable"

# Pousser le tag
git push origin v1.0.0

# Pousser tous les tags d'un coup
git push origin --tags

# Lister les tags
git tag

# Voir les détails d'un tag
git show v1.0.0
```

**Règle :** chaque tag doit être accompagné d'une release sur la plateforme (GitHub/GitLab : *Releases → New release*).

---

## Merge Requests / Pull Requests

Une **Merge Request** (GitLab) ou **Pull Request** (GitHub, Azure DevOps) est la procédure formelle pour intégrer une branche de travail dans `dev` ou `main`.

**Avant d'ouvrir une PR :**

1. S'assurer que les tests passent localement
2. Vérifier que le linting passe
3. Synchroniser sa branche avec `dev` : `git fetch origin && git rebase origin/dev`
4. Résoudre les conflits s'il y en a

**Bonnes pratiques :**

- Désigner les **bons reviewers** — au moins une personne qui connaît le code touché
- Écrire une description claire : **ce que fait la PR** et **pourquoi**
- Joindre une capture d'écran ou une démo si la PR touche une interface
- **Squash les commits** au merge si la branche contient des commits `wip` ou `fix typo` — l'historique de `dev` et `main` doit rester propre

---

## Issues

Les **Issues** tracent les demandes, les bugs et les tâches. Elles sont le point d'entrée de tout changement sur le projet.

**Quand créer une issue :**
- Un bug à corriger a été identifié
- Une nouvelle fonctionnalité est demandée
- Un problème technique doit être discuté avant d'être traité

**Bonne pratique :** associer chaque branche de travail et chaque PR à l'issue correspondante. Sur GitHub et GitLab, mentionner `Closes #42` dans le message de PR ferme automatiquement l'issue au merge.
